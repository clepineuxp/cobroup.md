# COBROUP — Backlog obligatorio de seguridad para firma JWT simétrica

> **Estado:** VIGENTE  
> **Decisión vinculante:** `DECISION_AUTENTICACION_JWT.md`  
> **Algoritmo autorizado:** exclusivamente `HS256`  
> **Condición:** los controles marcados como bloqueantes deben estar implementados y auditados antes de producción.

| # | Control obligatorio | Evidencia de aceptación | Condición |
|---|---|---|---|
| SYM-01 | Generación criptográfica y separación | Dev, QA y Prod poseen claves CSPRNG distintas, Base64, de 32 bytes o más; no hay frases humanas ni reutilización. | Bloquea producción |
| SYM-02 | Custodia GitHub/Kubernetes | Los valores reales existen solo como secretos; no aparecen en Git, ConfigMap, imagen, frontend, artefactos, render ni logs. | Bloquea producción |
| SYM-03 | Distribución mínima | Solo BackendAdmin y Backend pueden leer el Secret del ambiente; RBAC, ServiceAccount y namespace impiden acceso de frontends u otras cargas. | Bloquea producción |
| SYM-04 | Validación y fail-fast | Ambos servicios rechazan secreto ausente, Base64 inválido o menos de 32 bytes; no hay valores predeterminados ni fallback. | Bloquea producción |
| SYM-05 | Algoritmo estricto | Emisión y validación permiten únicamente `HS256`; se rechazan `none`, RSA y cualquier otro HMAC. | Bloquea producción |
| SYM-06 | Emisor único lógico | Solo BackendAdmin contiene casos de uso/endpoints de emisión. Backend usa la clave exclusivamente para validar. | Bloquea producción |
| SYM-07 | Rotación y solapamiento | Clave activa y anteriores se identifican con `kid`; las anteriores solo validan durante una ventana acotada. Procedimientos de rollout, rollback y emergencia están probados. | Bloquea producción |
| SYM-08 | Consistencia multirréplica | Todas las réplicas de un ambiente usan el mismo conjunto activo/anterior y aceptan tokens emitidos por cualquier réplica autorizada. | Bloquea producción |
| SYM-09 | Observabilidad segura | Alertas de fallo de carga/rotación/firma sin registrar claves ni JWT completos; acceso y cambios de Secret son auditables. | Bloquea producción |
| SYM-10 | Gate CI/CD | El render/despliegue falla por secreto ausente o inválido, placeholders, exposición, configuración cruzada o algoritmo distinto de `HS256`. | Bloquea producción |

La aceptación de firma simétrica no elimina el riesgo: comprometer cualquiera de los dos backends puede exponer capacidad de firma. Este riesgo se mitiga con mínimo privilegio, aislamiento por ambiente, rotación, telemetría y respuesta a incidentes; no se transforma en un requisito futuro de RSA.
