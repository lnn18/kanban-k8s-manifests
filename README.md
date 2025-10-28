# Kanban Board - Despliegue en Kubernetes

Guía completa para desplegar una aplicación Kanban Board en Kubernetes con PostgreSQL, Backend API y Frontend web.

## Tabla de Contenidos

- [Arquitectura](#arquitectura)
- [Prerrequisitos](#prerrequisitos)
- [Preparación de Imágenes](#preparación-de-imágenes)
- [Despliegue Completo](#despliegue-completo)
- [Verificación y Acceso](#verificación-y-acceso)
- [Comprobaciones Rápidas](#comprobaciones-rápidas)
- [Referencias](#referencias)

## Arquitectura

La aplicación Kanban consta de 3 componentes:

- **Frontend (kanban-ui)**: Interfaz web React en puerto 80
- **Backend (kanban-app)**: API REST Spring Boot en puerto 8080
- **Base de Datos**: PostgreSQL 9.6 en puerto 5432

**Flujo de comunicación:**
```
Usuario → Frontend (80) → Backend (8080) → PostgreSQL (5432)
```

## Prerrequisitos

### Herramientas necesarias:
- `kubectl` (v1.20+)
- `docker` o `podman`
- Cluster Kubernetes:
  - **Local**: `kind`, `minikube`, o `k3d`
  - **Remoto**: Acceso configurado con `kubectl`

### Verificar instalación:
```bash
kubectl version --client
docker --version
```

## Preparación de Imágenes

Cargar imágenes locales (Kind)

Si tienes las imágenes construidas localmente:

```bash
# Construir imágenes (desde el repo kanban-board)
git clone https://github.com/braybaut/kanban-board.git
cd kanban-board

# Construir backend
docker build -t kanban-app:latest ./kanban-app

# Construir frontend  
docker build -t kanban-ui:latest ./kanban-ui

# Cargar en kind
kind load docker-image kanban-app:latest
kind load docker-image kanban-ui:latest 
```

## Despliegue Completo

### 1. Clonar repositorio con submódulos (si aplica)

```bash
git clone --recurse-submodules https://github.com/braybaut/kanban-board.git
cd kanban-board/k8s-manifests  # o la carpeta donde estén los manifiestos
```

### 2. Desplegar PostgreSQL primero

```bash
kubectl apply -f postgres-deployment.yaml
```

**¿Por qué primero?** La base de datos debe estar lista antes que el backend intente conectarse.

### 3. Desplegar Backend

```bash
kubectl apply -f backend-deployment.yaml
```

### 4. Desplegar Frontend

```bash
kubectl apply -f frontend-deployment.yaml
```

## Verificación y Acceso

### Verificar que todos los pods estén corriendo:

```bash
kubectl get pods
```

**Salida esperada:**
```
NAME                                      READY   STATUS    RESTARTS   AGE
kanban-postgres-deployment-xxx            1/1     Running   0          2m
kanban-app-deployment-xxx                 1/1     Running   0          1m
kanban-ui-deployment-xxx                  1/1     Running   0          1m
```

### Verificar servicios:

```bash
kubectl get svc
```

### Exponer la aplicación:

```bash
# Exponer frontend (puerto 8080 local → puerto 80 del servicio)
kubectl port-forward svc/kanban-ui 8080:80

# En otra terminal, exponer backend para debugging (opcional)
kubectl port-forward svc/kanban-app 8081:8080
```

### Acceder a la aplicación:

Abrir navegador en: **http://localhost:8080**



## Referencias

- **Repositorio original:** https://github.com/wkrzywiec/kanban-board
- **Fork utilizado:** https://github.com/braybaut/kanban-board
- **Documentación Kubernetes:** https://kubernetes.io/docs/
- **Kind:** https://kind.sigs.k8s.io/
- **Minikube:** https://minikube.sigs.k8s.io/

---

