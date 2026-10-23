# Runbook operacional — Rotación de claves JWT simétricas (HS256) por ambiente

> **Estado:** VIGENTE  
> **Alcance:** `MS.Cobroup.BackendAdmin`, `MS.Cobroup.Backend`, `cobroup.environments`  
> **Algoritmo:** Exclusivamente `HS256` (`SecurityAlgorithms.HmacSha256`)  
> **Normativa:** `DECISION_AUTENTICACION_JWT.md` y `BACKLOG_SEGURIDAD_FIRMA_SIMETRICA.md` (`SYM-07`)  

---

## 1. Principios del diseño de rotación

1. **Identificador de versión (`kid`):**
   Cada clave simétrica tiene un identificador alfanumérico no secreto (ej. `k1`, `k2`, `2026-q4`).
   Los nuevos tokens generados por `MS.Cobroup.BackendAdmin` se firman con la clave correspondiente al `Jwt:ActiveKeyId`.
2. **Conjunto de claves anteriores para validación:**
   Ambos backends (`BackendAdmin` y `Backend`) soportan la colección `Jwt:PreviousValidationKeys:{kid}`, permitiendo validar tokens emitidos previamente sin permitir que una clave anterior firme nuevos tokens.
3. **Secreto simétrico:**
   Generado con CSPRNG criptográficamente seguro (mínimo 32 bytes / 256 bits), codificado en Base64.
   Nunca derivado de contraseñas ni frases humanas.
4. **Tratamiento de Refresh Tokens:**
   Los refresh tokens son JWT con `typ=refresh` y firma `HS256`. Al rotar la clave:
   - Durante la ventana de transición, la clave anterior en `Jwt:PreviousValidationKeys` valida los refresh tokens existentes cuando el cliente invoca `POST /api/v1/auth/refresh`.
   - La respuesta del refresh emite un **nuevo refresh token firmado con la clave activa nueva**, reemitiendo la sesión atómicamente hacia la nueva clave sin desconectar al usuario.
   - En rotaciones de emergencia (compromiso), se revoca inmediatamente la familia o se purgan las sesiones anteriores, forzando un nuevo login seguro.

---

## 2. Procedimiento de rotación programada (Cero downtime)

### Fase 1: Generación de la nueva clave criptográfica
En una estación administrativa segura o runner aislado:

```powershell
# Generar 32 bytes criptográficamente seguros en Base64
$bytes = New-Object byte[] 32
[System.Security.Cryptography.RandomNumberGenerator]::Create().GetBytes($bytes)
$newKeyBase64 = [Convert]::ToBase64String($bytes)
```
*(No guardar el valor en texto plano ni imprimirlo en consolas compartidas).*

Definir el nuevo `kid`, por ejemplo `k2`.

### Fase 2: Instalación de la nueva clave como secundaria/anterior en ambos backends
Para el ambiente objetivo (ej. `dev`, `qa` o `prod`):
1. En GitHub Environment Secrets del repositorio `cobroup.environments`:
   - Conservar la clave activa actual `DEV_JWT_SIGNING_KEY_BASE64` (asociada a `k1`).
   - Configurar en la configuración de la aplicación `Jwt__PreviousValidationKeys__k2` con el nuevo secreto, o preparar la clave activa nueva manteniendo `k1` como clave previa.

**Estrategia recomendada de transición:**
- Clave activa emisora en BackendAdmin: `k1`.
- Clave secundaria de validación en BackendAdmin y Backend: `k2`.
- Desplegar ambos backends y verificar que reconozcan `k2` como clave de validación.

### Fase 3: Activación coordinada de la nueva clave emisora
1. Actualizar el secreto GitHub del ambiente (`{ENV}_JWT_SIGNING_KEY_BASE64`) con el valor de la clave `k2`.
2. En los ConfigMaps (`admin-backend-config` y `backend-config`):
   - Establecer `Jwt__ActiveKeyId: "k2"`.
   - Mapear la clave `k1` anterior en `Jwt__PreviousValidationKeys__k1` en los Secrets de ambos backends.
3. Desplegar primero `backend` (validador operativo) y después `admin-backend` (emisor):
   - `backend` puede validar tokens tanto con `k2` como con `k1`.
   - `admin-backend` comienza a emitir tokens con `kid: "k2"`.
4. Todos los nuevos logins y refresh emiten tokens firmados con `k2`.
5. Los access tokens anteriores firmados con `k1` siguen siendo aceptados por ambos backends gracias a `PreviousValidationKeys`.

### Fase 4: Ventana de solapamiento
- **Duración:** TTL máximo de access token (15 minutos) + tolerancia máxima de reloj (5 minutos skew) = **20 minutos**.
- Durante estos 20 minutos:
  - Los access tokens antiguos con `k1` expiran naturalmente.
  - Los usuarios que interactúan refrescan su sesión y reciben tokens con `k2`.

### Fase 5: Retiro verificable de la clave anterior
1. Transcurridos los 20 minutos (o ventana extendida acordada para refresh si no se opta por revocación):
   - Retirar `Jwt__PreviousValidationKeys__k1` de los Secrets y ConfigMaps.
2. Desplegar `admin-backend` y `backend`.
3. **Verificación:** Probar que un token antiguo firmado con `k1` es rechazado inmediatamente con `401 Unauthorized` (`Unknown kid` o firma no confiable).

---

## 3. Procedimiento de rotación de emergencia (Por compromiso de clave)

Si se sospecha o confirma que el secreto simétrico de un ambiente ha sido expuesto:

1. **Generación inmediata:** Generar un nuevo secreto CSPRNG de 32 bytes y asignar un nuevo `kid` (ej. `k-emergency-YYYYMMDD`).
2. **Sustitución atómica sin solapamiento:**
   - Actualizar inmediatamente `{ENV}_JWT_SIGNING_KEY_BASE64` en GitHub Secrets con el nuevo valor.
   - En ConfigMaps, establecer `Jwt__ActiveKeyId: "k-emergency-YYYYMMDD"`.
   - **NO** configurar claves anteriores en `PreviousValidationKeys`. La clave comprometida se descarta de inmediato.
3. **Revocación de sesiones en base de datos:**
   - Ejecutar script de revocación en la base de datos de identidad para marcar todos los refresh tokens activos como revocados (`revoked_at = NOW()`).
4. **Reinicio/Rollout de pods:**
   ```bash
   kubectl rollout restart deployment/admin-backend -n cobroup-{env}
   kubectl rollout restart deployment/backend -n cobroup-{env}
   ```
5. **Efecto:**
   - Todos los access tokens anteriores quedan invalidados de inmediato.
   - Todos los refresh tokens quedan invalidados; ningún actor malicioso puede reemitir credenciales.
   - Los usuarios legítimos reciben `401 Unauthorized` y deben autenticarse nuevamente con usuario y contraseña.

---

## 4. Procedimiento de Rollback (Fallo operativo durante despliegue)

Si durante el paso de activación (Fase 3) se detectan anomalías de conectividad o errores de configuración:

1. Revertir el valor de `{ENV}_JWT_SIGNING_KEY_BASE64` y `Jwt__ActiveKeyId` al `kid` previo (`k1`).
2. Desplegar `admin-backend` y `backend`.
3. Puesto que la clave `k1` nunca dejó de ser válida durante la ventana de solapamiento, el servicio recupera la operatividad sin interrupción de sesiones.
4. Investigar la causa en logs seguros (los logs nunca mostrarán el secreto).

---

## 5. Auditoría y Registro seguro

- Los backends y el script de renderizado registran únicamente eventos criptográficos seguros:
  - `[OK] Injected Jwt__SigningKeyBase64 into {file} (redacted)`
  - `Authentication failed: kid 'k1' is not recognized or untrusted`
- **PROHIBIDO:** Registrar en logs de aplicación, stdout, stderr, excepciones o GitHub Actions el valor del secreto, subcadenas o representaciones reversibles.
