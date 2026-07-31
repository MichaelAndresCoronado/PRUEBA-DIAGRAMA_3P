# 🍷 CavaLocal

> **Marketplace de vinos basado en Microservicios con Kubernetes, MongoDB, RabbitMQ y Auditoría Distribuida**

## Universidad de las Fuerzas Armadas ESPE

**Autores**

- Michael Coronado
- Kevin Panata

---

# Tecnologías

- NestJS
- MongoDB
- RabbitMQ
- Kubernetes
- Docker
- Nginx
- Vanilla JavaScript
- Server-Sent Events (SSE)

---

# Arquitectura

La solución implementa una arquitectura de microservicios desplegada sobre Kubernetes.

## Diagrama de Arquitectura (Pods, Services y Flujo de Red)

```mermaid
flowchart TB

classDef user fill:#f1f8ff,stroke:#0366d6,color:#000;
classDef ingress fill:#fff8f2,stroke:#ea4aaa,color:#000;
classDef frontend fill:#e1f5fe,stroke:#0288d1,color:#000;
classDef backend fill:#e8f5e9,stroke:#388e3c,color:#000;
classDef rabbit fill:#ffebee,stroke:#d32f2f,color:#000;
classDef mongo fill:#dcedc8,stroke:#4CAF50,color:#000;
classDef audit fill:#fff3e0,stroke:#fb8c00,color:#000;
classDef service fill:#ffffff,stroke:#6c757d,color:#000;

U([Cliente]):::user
I[Ingress Controller<br/>conjunta3p.espe.edu.ec]:::ingress

U --> I

subgraph Kubernetes Cluster
direction TB

  subgraph Frontend Layer
  direction LR
  FS[frontend-svc]:::service --> FP[Frontend<br/>Nginx + JS]:::frontend
  end

  subgraph Backend Layer
  direction LR
  BS[backend-svc]:::service --> BP[Backend<br/>NestJS]:::backend
  end

  subgraph Audit Layer
  direction LR
  AS[audit-svc]:::service --> A1[Audit Replica 1]:::audit
  AS --> A2[Audit Replica 2]:::audit
  end

  subgraph Messaging and Data
  direction LR
  RS[rabbitmq-svc]:::service --> RB[(RabbitMQ)]:::rabbit
  MS[mongodb-svc]:::service --> DB[(MongoDB)]:::mongo
  end

  I --> FS
  I --> BS
  I --> AS
  BP -->|REST| DB
  BP -->|Publish Event| RB
  A1 -->|Consume| RB
  A2 -->|Consume| RB
  A1 -->|Guardar Auditoría| DB
  A2 -->|Guardar Auditoría| DB
  A1 -->|SSE| FP
  A2 -->|SSE| FP

end
```

# Flujo de Auditoría

1. Cliente realiza una operación.
2. Backend procesa la solicitud.
3. Backend guarda información en MongoDB.
4. Backend publica un evento en RabbitMQ.
5. Servicio de Auditoría consume el evento.
6. Guarda la auditoría en MongoDB.
7. Envía el evento mediante SSE al Dashboard.

---

# Despliegue Paso a Paso

## Requisitos

- Docker Desktop
- Minikube o Kind
- kubectl
- Git
- PowerShell

## Opción 1: Minikube

1. Clona el repositorio.

```bash
git clone <url>
cd CavaLocal
```

2. Inicia Minikube y habilita los complementos necesarios.

```powershell
minikube start --driver=docker
minikube addons enable ingress
minikube addons enable dashboard
```

3. Conecta Docker al clúster de Minikube para construir las imágenes dentro del entorno local.

```powershell
& minikube -p minikube docker-env --shell powershell | Invoke-Expression
```

4. Construye las imágenes del proyecto.

```powershell
docker build -t cavalocal-backend ./backend
docker build -t cavalocal-audit ./audit-service
docker build -t cavalocal-frontend ./web
```

5. Aplica los manifiestos de Kubernetes.

```powershell
kubectl apply -f k8s/
```

6. Verifica el estado del clúster.

```powershell
kubectl get pods
kubectl get svc
kubectl get ingress
```

Todos los Pods deben quedar en estado **Running** y el Ingress debe resolver el dominio configurado.

## Opción 2: Kind

1. Crea el clúster con Kind.

```powershell
kind create cluster --name cavalocal
```

2. Carga las imágenes Docker en el clúster.

```powershell
docker build -t cavalocal-backend ./backend
docker build -t cavalocal-audit ./audit-service
docker build -t cavalocal-frontend ./web
kind load docker-image cavalocal-backend --name cavalocal
kind load docker-image cavalocal-audit --name cavalocal
kind load docker-image cavalocal-frontend --name cavalocal
```

3. Aplica los manifiestos y verifica el funcionamiento con `kubectl get pods`, `kubectl get svc` y `kubectl get ingress`.

---

# Variables de Entorno

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cavalocal-config
data:
  RABBITMQ_HOST: "rabbitmq"
  RABBITMQ_PORT: "5672"
  MONGO_HOST: "mongodb"
  MONGO_PORT: "27017"
  MONGO_DB: "audit_db"
  POSTGRES_HOST: "postgres"
  POSTGRES_PORT: "5432"
  POSTGRES_DB: "cavalocal"
```

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: cavalocal-secrets
type: Opaque
data:
  RABBITMQ_DEFAULT_USER: "Z3Vlc3Q="
  RABBITMQ_DEFAULT_PASS: "Z3Vlc3Q="
  MONGO_INITDB_ROOT_USERNAME: "YWRtaW4="
  MONGO_INITDB_ROOT_PASSWORD: "YWRtaW4="
  POSTGRES_USER: "YWRtaW4="
  POSTGRES_PASSWORD: "YWRtaW4="
  JWT_SECRET: "bXlfc3VwZXJfc2VjcmV0X2tleQ=="
```

## Inyección segura en los Deployments

Las variables no sensibles deben ir en el `ConfigMap` y las credenciales en `Secret`. Luego se inyectan en cada `Deployment` con `envFrom`.

```yaml
envFrom:
  - configMapRef:
      name: cavalocal-config
  - secretRef:
      name: cavalocal-secrets
```

Si prefieres crear el `Secret` desde consola en vez de escribirlo a mano, puedes usar:

```powershell
kubectl create secret generic cavalocal-secrets `
  --from-literal=RABBITMQ_DEFAULT_USER=guest `
  --from-literal=RABBITMQ_DEFAULT_PASS=guest `
  --from-literal=MONGO_INITDB_ROOT_USERNAME=admin `
  --from-literal=MONGO_INITDB_ROOT_PASSWORD=admin `
  --from-literal=POSTGRES_USER=admin `
  --from-literal=POSTGRES_PASSWORD=admin `
  --from-literal=JWT_SECRET=my_super_secret_key
```

## MongoDB

```
MONGO_INITDB_ROOT_USERNAME
MONGO_INITDB_ROOT_PASSWORD
MONGO_URL
```

## RabbitMQ

```
RABBITMQ_DEFAULT_USER
RABBITMQ_DEFAULT_PASS
RABBITMQ_URL
```

## Backend

```
JWT_SECRET
GOOGLE_CLIENT_ID
MAIL_USER
MAIL_APP_PASSWORD
POSTGRES_HOST
POSTGRES_PORT
POSTGRES_DB
POSTGRES_USER
POSTGRES_PASSWORD
```

# Secrets

Todas las credenciales se almacenan mediante **Kubernetes Secrets**.

Ejemplo:

```bash
echo -n "mi_password" | base64
```

Colocar los valores codificados dentro de `k8s/secrets.yaml`.

---

# Configuración del archivo hosts

Para acceder al dominio `conjunta3p.espe.edu.ec`, agrega la IP del clúster al archivo `hosts`.

Obtener IP:

```bash
minikube ip
```

Editar:

Windows

```
C:\Windows\System32\drivers\etc\hosts
```

Linux/macOS

```
/etc/hosts
```

Agregar:

```
IP_MINIKUBE conjunta3p.espe.edu.ec
```

En Windows, abre el archivo como administrador. En Linux/macOS usa permisos de superusuario.

---

# Accesos

- Dashboard: http://conjunta3p.espe.edu.ec/dashboard
- API: http://conjunta3p.espe.edu.ec/api
- Auditoría: http://conjunta3p.espe.edu.ec/api/audit

---

# Despliegue Automático

El proyecto incluye el script **deploy.ps1**, el cual automatiza el flujo completo para Minikube.

## Ejecutar

```powershell
.\deploy.ps1
```

Si PowerShell bloquea la ejecución:

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass
.\deploy.ps1
```

## El script realiza

- Inicia Minikube.
- Habilita Ingress.
- Construye imágenes Docker.
- Aplica todos los manifiestos.
- Espera que los Pods estén Running.
- Presenta el sistema completamente funcional.

## Verificación

```bash
kubectl get pods
kubectl get services
kubectl get ingress
```

Si todos los Pods están en **Running**, el sistema estará listo para utilizarse.
