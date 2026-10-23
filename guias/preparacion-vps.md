# Guía Maestra de Preparación de VPS y GitOps para COBROUP

> **Versión:** 2.3.0  
> **Fecha:** 2026-10-22  
> **Plataforma objetivo:** VPS Linux (Ubuntu 22.04 / 24.04 LTS), K3s con Ingress-NGINX y Cert-Manager  
> **Repositorio de Manifiestos y Secretos:** [https://github.com/clepineuxp/cobroup.enviroments.git](https://github.com/clepineuxp/cobroup.enviroments.git)  
> **Ambientes soportados:** `cobroup-dev` | `cobroup-qa` | `cobroup-prod`  
> **Arquitectura:** GitOps desacoplado mediante `cobroup.enviroments` y pipelines independientes por microservicio  
> **Registro OCI:** GitHub Container Registry (`ghcr.io`)  
> **Despliegue automatizado:** Exportación de manifiestos SCP + Rollout progresivo vía SSH

---

## 1. Visión general de la arquitectura de despliegue (Modelo SaviaUp)

El ecosistema **COBROUP** implementa una arquitectura **GitOps desacoplada y robusta**, basada en el estándar probado de `saviaup.environments`:

```
┌────────────────────────────────────────────────────────────────────────┐
│                      REPOSITORIOS DE CÓDIGO FUENTE                     │
│  • MS.Cobroup.Backend       (API Operaciones)                          │
│  • MS.Cobroup.BackendAdmin  (Autoridad de identidad JWT)               │
│  • MF.cobroup-frontend      (PWA Cobrador / Recolector)                │
│  • MF.cobroup-frontend-admin(PWA Portal Administrativo)                │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Push a rama dev/qa/prod
                                    ▼
                    ┌───────────────────────────────┐
                    │     Pipeline docker-ghcr.yml  │
                    │  1. Tests automatizados 100%  │
                    │  2. Build Docker multi-stage  │
                    │  3. Push a ghcr.io            │
                    └───────────────┬───────────────┘
                                    │ SSH (kubectl apply & rollout restart)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                   REPOSITORIO cobroup.enviroments                      │
│   https://github.com/clepineuxp/cobroup.enviroments.git                │
│  • dev/, qa/, prod/ (Kustomize, Ingress, Secrets, ConfigMaps)          │
│  • bootstrap/ (cobroup-infra-bootstrap.yaml)                           │
│  • deploy-dev.yml, deploy-qa.yml, deploy-prod.yml                      │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ Push a rama dev/qa/prod
                                    ▼
                    ┌───────────────────────────────┐
                    │ Pipeline deploy-<env>.yml     │
                    │  1. Sustitución Python secrets│
                    │  2. SCP a /opt/cobroup/deploys│
                    └───────────────┬───────────────┘
                                    │ SCP (Manifiestos renderizados)
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                              VPS LINUX                                 │
│  Directorios de Manifiestos:                                           │
│  ├── /opt/cobroup/deploys/bootstrap/ (cobroup-infra-bootstrap.yaml)    │
│  ├── /opt/cobroup/deploys/dev/       (backend, admin-backend, fronts)  │
│  ├── /opt/cobroup/deploys/qa/                                          │
│  └── /opt/cobroup/deploys/prod/                                        │
│                                                                        │
│  Clúster K3s:                                                          │
│  ├── Ingress-NGINX Controller + Let's Encrypt (Cert-Manager)           │
│  ├── Namespace cobroup-dev   (dev.cobroup.com, dev-api.cobroup.com...) │
│  ├── Namespace cobroup-qa    (qa.cobroup.com, qa-api.cobroup.com...)   │
│  └── Namespace cobroup-prod  (cobroup.com, api.cobroup.com [2 réplicas│
│                                                                        │
│  Base de Datos: PostgreSQL 16 (Schemas: identity | operations)         │
└────────────────────────────────────────────────────────────────────────┘
```

### Decisión de identidad aplicable al despliegue

La autoridad utiliza exclusivamente access y refresh JWT bajo `../DECISION_AUTENTICACION_JWT.md`. BackendAdmin firma con `HS256` y Backend operativo valida localmente con el mismo secreto del ambiente. Dev, QA y Prod deben usar valores CSPRNG distintos, Base64, de 32 bytes o más, administrados como secretos de GitHub/Kubernetes desde `cobroup.environments`. La falta de clave, Base64 inválido, longitud insuficiente, `kid`, issuer o audiences debe impedir el arranque. Quedan prohibidos valores predeterminados, RSA, JWKS, algoritmos alternativos y fallbacks. Los manifiestos existentes deben adecuarse y reauditarse antes de desplegar.

---

## 2. Lo que debes hacer en el repositorio `cobroup.enviroments`

El repositorio [https://github.com/clepineuxp/cobroup.enviroments.git](https://github.com/clepineuxp/cobroup.enviroments.git) es el **núcleo de infraestructura y secretos** de COBROUP. Sigue estos pasos para configurarlo:

### Paso A: Subir las ramas iniciales a GitHub
En tu máquina local, el repositorio ya tiene configurado el remoto `origin`:

```bash
cd D:\Proyectos\cobroup\cobroup.enviroments

# 1. Subir rama main
git push -u origin main

# 2. Subir rama dev
git push -u origin dev

# 3. Subir rama qa
git push -u origin qa
```

---

### Paso B: Crear el Entorno `cobroup` en GitHub
1. Abre tu navegador y ve a [https://github.com/clepineuxp/cobroup.enviroments](https://github.com/clepineuxp/cobroup.enviroments).
2. Ve a **Settings** ➔ **Environments**.
3. Haz clic en **New environment**.
4. Escribe el nombre exacto: `cobroup` y guarda.
> **Por qué:** Los pipelines `deploy-dev.yml`, `deploy-qa.yml` y `deploy-prod.yml` declaran `environment: cobroup`.

---

### Paso C: Cargar los Secretos en `cobroup.enviroments`
En **Settings** ➔ **Secrets and variables** ➔ **Actions** (puedes crearlos en **Repository secrets** o dentro del Environment `cobroup`):

| Nombre del Secreto | Tipo | Descripción | Ejemplo de Valor |
|---|:---:|---|---|
| `SSH_HOST` | Secreto | IP pública de la VPS | `198.51.100.25` |
| `SSH_USER` | Secreto | Usuario dedicado en la VPS | `deployer` |
| `SSH_KEY` | Secreto | Clave privada SSH Ed25519 completa | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `SSH_PORT` | Secreto | Puerto SSH del servidor | `22` |
| `GHCR_TOKEN` | Secreto | GitHub Personal Access Token (`read:packages`) | `ghp_xxxxxxxxxxxx` |
| `DEV_OPERATIONSDATABASE_CONNECTIONSTRING` | Secreto | Cadena PostgreSQL esquema operations (Dev) | `Host=10.42.0.1;Database=cobroup_dev;SearchPath=operations;Username=cobroup_dev_user;Password=...` |
| `DEV_IDENTITYDATABASE_CONNECTIONSTRING` | Secreto | Cadena PostgreSQL esquema identity (Dev) | `Host=10.42.0.1;Database=cobroup_dev;SearchPath=identity;Username=cobroup_dev_user;Password=...` |
| `DEV_JWT_SIGNING_KEY_BASE64` | Secreto | Clave HS256 CSPRNG exclusiva de Dev; se inyecta en ambos backends | `<BASE64_32_BYTES_OR_MORE_DEV>` |
| `QA_OPERATIONSDATABASE_CONNECTIONSTRING` | Secreto | Cadena PostgreSQL esquema operations (QA) | `Host=10.42.0.1;Database=cobroup_qa;SearchPath=operations;Username=cobroup_qa_user;Password=...` |
| `QA_IDENTITYDATABASE_CONNECTIONSTRING` | Secreto | Cadena PostgreSQL esquema identity (QA) | `Host=10.42.0.1;Database=cobroup_qa;SearchPath=identity;Username=cobroup_qa_user;Password=...` |
| `QA_JWT_SIGNING_KEY_BASE64` | Secreto | Clave HS256 CSPRNG exclusiva de QA; se inyecta en ambos backends | `<BASE64_32_BYTES_OR_MORE_QA>` |
| `PROD_OPERATIONSDATABASE_CONNECTIONSTRING` | Secreto | Cadena PostgreSQL esquema operations (Prod) | `Host=10.42.0.1;Database=cobroup_prod;SearchPath=operations;Username=cobroup_prod_user;Password=...` |
| `PROD_IDENTITYDATABASE_CONNECTIONSTRING` | Secreto | Cadena PostgreSQL esquema identity (Prod) | `Host=10.42.0.1;Database=cobroup_prod;SearchPath=identity;Username=cobroup_prod_user;Password=...` |
| `PROD_JWT_SIGNING_KEY_BASE64` | Secreto | Clave HS256 CSPRNG exclusiva de Prod; se inyecta en ambos backends | `<BASE64_32_BYTES_OR_MORE_PROD>` |

---

### Paso D: Cargar las Variables en `cobroup.enviroments`
En **Settings** ➔ **Secrets and variables** ➔ **Actions** ➔ pestaña **Variables**:

| Nombre de Variable | Valor | Descripción |
|---|---|---|
| `GHCR_USERNAME` | `clepineuxp` | Usuario GitHub propietario de los paquetes OCI |

---

### Paso E: Ejecutar la Exportación Inicial de Manifiestos
Una vez creados los secretos:
1. En GitHub ve a la pestaña **Actions** de `cobroup.enviroments`.
2. Selecciona **Export Dev Manifests via SSH** y haz clic en **Run workflow** sobre la rama `dev`.
3. El pipeline:
   - Renderizará los secretos en los archivos `.yaml` con Python.
   - Creará en la VPS `/opt/cobroup/deploys/dev` y `/opt/cobroup/deploys/bootstrap`.
   - Transferirá todos los manifiestos listos para K3s mediante SCP.

---

## 3. Preparación paso a paso de la VPS (desde cero)

Dado que en la VPS únicamente está instalado el clúster con **Ingress-NGINX**, ejecuta los siguientes pasos como usuario `root` (o con `sudo`).

---

### Paso 1: Crear el usuario dedicado para despliegues (`deployer`)

```bash
# 1. Crear el usuario deployer sin contraseña interactiva
sudo adduser --disabled-password --gecos "" deployer

# 2. Agregar al grupo sudo
sudo usermod -aG sudo deployer

# 3. Crear directorio .ssh
sudo mkdir -p /home/deployer/.ssh
sudo chmod 700 /home/deployer/.ssh
```

---

### Paso 2: Generar y configurar el par de llaves SSH

En tu equipo local:
```bash
# Generar par de llaves Ed25519 (Enter a todo)
ssh-keygen -t ed25519 -C "github-actions-cobroup" -f ~/.ssh/cobroup_deployer
```

En la VPS, autoriza la clave pública:
```bash
echo "PEGA_AQUÍ_EL_CONTENIDO_DE_cobroup_deployer.pub" | sudo tee -a /home/deployer/.ssh/authorized_keys
sudo chmod 600 /home/deployer/.ssh/authorized_keys
sudo chown -R deployer:deployer /home/deployer/.ssh
```

---

### Paso 3: Configurar acceso a `kubectl` para `deployer`

```bash
# 1. Crear directorio .kube
sudo mkdir -p /home/deployer/.kube

# 2. Copiar kubeconfig
sudo cp /etc/rancher/k3s/k3s.yaml /home/deployer/.kube/config

# 3. Asignar permisos a deployer
sudo chown -R deployer:deployer /home/deployer/.kube
sudo chmod 600 /home/deployer/.kube/config

# 4. Verificar funcionamiento
sudo -u deployer kubectl get nodes
```

---

### Paso 4: Crear la estructura de directorios de despliegue (`/opt/cobroup`)

```bash
sudo mkdir -p /opt/cobroup/deploys/bootstrap
sudo mkdir -p /opt/cobroup/deploys/dev
sudo mkdir -p /opt/cobroup/deploys/qa
sudo mkdir -p /opt/cobroup/deploys/prod
sudo chown -R deployer:deployer /opt/cobroup
sudo chmod -R 775 /opt/cobroup
```

---

### Paso 5: Bootstrap de Namespaces en K3s

```bash
kubectl create namespace cobroup-dev --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace cobroup-qa --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace cobroup-prod --dry-run=client -o yaml | kubectl apply -f -
```

---

### Paso 6: Generar Personal Access Token (PAT) de GitHub para GHCR

1. En GitHub ve a **Settings** -> **Developer Settings** -> **Personal access tokens** -> **Tokens (classic)**.
2. Clic en **Generate new token (classic)**.
3. Nombre: `cobroup-k3s-pull` con scope `read:packages`.
4. En la VPS crea el secreto `ghcr-secret` en los 3 namespaces:

```bash
export GH_USER="<TU_USUARIO_GITHUB>"
export GH_PAT="<TU_PAT_COPIADO>"

for NS in cobroup-dev cobroup-qa cobroup-prod; do
  echo "Configurando ghcr-secret en $NS..."
  kubectl create secret docker-registry ghcr-secret \
    --namespace="$NS" \
    --docker-server=ghcr.io \
    --docker-username="$GH_USER" \
    --docker-password="$GH_PAT" \
    --dry-run=client -o yaml | kubectl apply -f -
done
```

---

### Paso 7: Instalar Cert-Manager y ClusterIssuer Let's Encrypt

```bash
# 1. Instalar Cert-Manager v1.16.2
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml

# 2. Esperar que los pods de cert-manager estén activos
kubectl wait --namespace cert-manager --for=condition=ready pod --all --timeout=120s

# 3. Aplicar ClusterIssuer de Let's Encrypt para Ingress-NGINX
cat << 'EOF' | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@cobroup.com   # <-- REEMPLAZAR POR EMAIL REAL
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
      - http01:
          ingress:
            class: nginx
EOF
```

---

### Paso 8: Instalar y Configurar PostgreSQL 16

```bash
sudo apt update && sudo apt install -y postgresql postgresql-contrib
sudo systemctl enable --now postgresql

# Crear usuarios y bases de datos por ambiente
sudo -u postgres psql << 'EOF'
CREATE USER cobroup_dev_user WITH ENCRYPTED PASSWORD 'Cobroup_Dev_Password_2026!';
CREATE DATABASE cobroup_dev OWNER cobroup_dev_user;

CREATE USER cobroup_qa_user WITH ENCRYPTED PASSWORD 'Cobroup_Qa_Password_2026!';
CREATE DATABASE cobroup_qa OWNER cobroup_qa_user;

CREATE USER cobroup_prod_user WITH ENCRYPTED PASSWORD 'Cobroup_Prod_Password_2026!';
CREATE DATABASE cobroup_prod OWNER cobroup_prod_user;
EOF

# Crear esquemas desacoplados 'identity' y 'operations'
sudo -u postgres psql -d cobroup_dev << 'EOF'
CREATE SCHEMA IF NOT EXISTS identity AUTHORIZATION cobroup_dev_user;
CREATE SCHEMA IF NOT EXISTS operations AUTHORIZATION cobroup_dev_user;
EOF

sudo -u postgres psql -d cobroup_qa << 'EOF'
CREATE SCHEMA IF NOT EXISTS identity AUTHORIZATION cobroup_qa_user;
CREATE SCHEMA IF NOT EXISTS operations AUTHORIZATION cobroup_qa_user;
EOF

sudo -u postgres psql -d cobroup_prod << 'EOF'
CREATE SCHEMA IF NOT EXISTS identity AUTHORIZATION cobroup_prod_user;
CREATE SCHEMA IF NOT EXISTS operations AUTHORIZATION cobroup_prod_user;
EOF
```

> **Permitir conectividad desde pods:** En `/etc/postgresql/16/main/postgresql.conf` configura `listen_addresses = '*'` y en `pg_hba.conf` agrega `host all all 10.42.0.0/16 md5`. Reinicia con `sudo systemctl restart postgresql`.

---

### Paso 9: Mapeo DNS

Apunta los registros tipo **A** a la IP pública de la VPS:

| Ambiente | Hostname DNS | Destino | Servicio Enrutado |
|---|---|---|---|
| **Dev** | `dev.cobroup.com` | `<IP_VPS>` | PWA Recolector (`frontend`) |
| **Dev** | `dev-admin.cobroup.com` | `<IP_VPS>` | Portal Admin (`admin-frontend`) |
| **Dev** | `dev-api.cobroup.com` | `<IP_VPS>` | API Operaciones (`backend`) |
| **Dev** | `dev-auth.cobroup.com` | `<IP_VPS>` | API Identidad (`admin-backend`) |
| **QA** | `qa.cobroup.com` | `<IP_VPS>` | PWA Recolector (`frontend`) |
| **QA** | `qa-admin.cobroup.com` | `<IP_VPS>` | Portal Admin (`admin-frontend`) |
| **QA** | `qa-api.cobroup.com` | `<IP_VPS>` | API Operaciones (`backend`) |
| **QA** | `qa-auth.cobroup.com` | `<IP_VPS>` | API Identidad (`admin-backend`) |
| **Prod** | `cobroup.com` / `www.cobroup.com` | `<IP_VPS>` | PWA Recolector (`frontend`) |
| **Prod** | `admin.cobroup.com` | `<IP_VPS>` | Portal Admin (`admin-frontend`) |
| **Prod** | `api.cobroup.com` | `<IP_VPS>` | API Operaciones (`backend`) |
| **Prod** | `auth.cobroup.com` | `<IP_VPS>` | API Identidad (`admin-backend`) |

---

## 4. Configuración en los 4 Repositorios de Código Fuente

En cada uno de los repositorios de microservicios:
- `MS.Cobroup.Backend`
- `MS.Cobroup.BackendAdmin`
- `MF.cobroup-frontend`
- `MF.cobroup-frontend-admin`

Configurar únicamente estos 4 secretos en **Settings ➔ Secrets and variables ➔ Actions**:

| Secreto | Descripción | Valor |
|---|---|---|
| `SSH_HOST` | Dirección IP pública de la VPS | `&lt;IP_DE_TU_VPS&gt;` |
| `SSH_USER` | Usuario dedicado | `deployer` |
| `SSH_KEY` | Clave privada SSH Ed25519 completa | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `SSH_PORT` | Puerto SSH del servidor | `22` |

---

## 5. Secuencia de Puesta en Marcha

1. **Subir `cobroup.enviroments` a GitHub** y correr el workflow `Export Dev Manifests via SSH`.
2. **En la VPS, aplicar el bootstrap y el primer despliegue declarativo:**
   ```bash
   kubectl apply -f /opt/cobroup/deploys/bootstrap/cobroup-infra-bootstrap.yaml
   kubectl apply -k /opt/cobroup/deploys/dev
   ```
3. **Hacer push a `dev` en los microservicios:**
   El pipeline `docker-ghcr.yml` de cada repositorio compilará, subirá la imagen a GHCR y ejecutará `kubectl rollout restart deployment/<component> -n cobroup-dev` con verificación automática de pods activos.
