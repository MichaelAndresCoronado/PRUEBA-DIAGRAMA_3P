# 🍷 CavaLocal

> **Marketplace de vinos basado en Microservicios con Kubernetes, MongoDB, RabbitMQ y Auditoría Distribuida**

## Universidad de las Fuerzas Armadas ESPE

**Autores**

- Michael Andrés Coronado Achig — Infraestructura, Kubernetes, Docker y DevOps
- Kevin — Desarrollo Backend

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
graph TD

classDef user fill:#f1f8ff,stroke:#0366d6,color:#000;
classDef ingress fill:#fff8f2,stroke:#ea4aaa,color:#000;
classDef frontend fill:#e1f5fe,stroke:#0288d1,color:#000;
classDef backend fill:#e8f5e9,stroke:#388e3c,color:#000;
classDef rabbit fill:#ffebee,stroke:#d32f2f,color:#000;
classDef mongo fill:#dcedc8,stroke:#4CAF50,color:#000;
classDef audit fill:#fff3e0,stroke:#fb8c00,color:#000;

U([Cliente]):::user
I[Ingress Controller<br/>conjunta3p.espe.edu.ec]:::ingress

U --> I

subgraph Kubernetes Cluster

FS[frontend-svc]:::frontend
BS[backend-svc]:::backend
AS[audit-svc]:::audit
RS[rabbitmq-svc]:::rabbit
MS[mongodb-svc]:::mongo

FP[Frontend<br/>Nginx + JS]:::frontend
BP[Backend<br/>NestJS]:::backend
A1[Audit Replica 1]:::audit
A2[Audit Replica 2]:::audit
RB[(RabbitMQ)]:::rabbit
DB[(MongoDB)]:::mongo

I --> FS
I --> BS
I --> AS

FS --> FP
BS --> BP
AS --> A1
AS --> A2

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
- Minikube
- kubectl
- Git
- PowerShell

## Clonar

```bash
git clone <url>
cd CavaLocal
```

## Levantar Minikube

```bash
minikube start --driver=docker
minikube addons enable ingress
minikube addons enable dashboard
```

## Construir imágenes

```bash
eval $(minikube docker-env)

docker build -t cavalocal-backend ./backend
docker build -t cavalocal-audit ./audit-service
docker build -t cavalocal-frontend ./web
```

## Aplicar manifiestos

```bash
kubectl apply -f k8s/
```

## Verificar funcionamiento

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```

Todos los Pods deben encontrarse en estado **Running**.

---

# Variables de Entorno

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

---

# Accesos

- Dashboard: http://conjunta3p.espe.edu.ec/dashboard
- API: http://conjunta3p.espe.edu.ec/api
- Auditoría: http://conjunta3p.espe.edu.ec/api/audit

---

# Despliegue Automático

El proyecto incluye el script **deploy.ps1**, el cual automatiza todo el proceso.

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
