# Kanban Board - Despliegue en Kubernetes

Guía completa para desplegar una aplicación Kanban Board en Kubernetes con PostgreSQL, Backend API y Frontend web.

## Tabla de Contenidos

- [Arquitectura](#arquitectura)
- [Prerrequisitos](#prerrequisitos)
- [Despliegue Rápido](#despliegue-rápido)
- [Preparación del Entorno](#preparación-del-entorno)
- [Configuración del Cluster](#configuración-del-cluster)
- [Preparación de Imágenes](#preparación-de-imágenes)
- [Despliegue Completo](#despliegue-completo)
- [Verificación y Acceso](#verificación-y-acceso)
- [Comprobaciones Rápidas](#comprobaciones-rápidas)
- [Troubleshooting](#troubleshooting)
- [Limpieza](#limpieza)
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

## Despliegue Rápido

**6 comandos esenciales para desplegar:**

```bash
# 1. Crear namespace
kubectl create namespace kanban

# 2. Aplicar manifiestos
kubectl apply -f . -n kanban

# 3. Esperar que los pods estén listos
kubectl wait --for=condition=ready pod -l app=kanban-postgres -n kanban --timeout=300s

# 4. Verificar despliegue
kubectl get pods -n kanban

# 5. Exponer frontend
kubectl port-forward svc/kanban-ui 8080:80 -n kanban &

# 6. Acceder a la aplicación
echo "Aplicación disponible en: http://localhost:8080"
```

## Preparación del Entorno

### Opción A: Cluster Local con Kind

```bash
# Instalar kind (si no está instalado)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Crear cluster
kind create cluster --name kanban-cluster

# Verificar contexto
kubectl config current-context
```

### Opción B: Cluster Local con Minikube

```bash
# Iniciar minikube
minikube start --driver=docker

# Configurar contexto
kubectl config use-context minikube
```

### Opción C: Cluster Remoto

```bash
# Listar contextos disponibles
kubectl config get-contexts

# Cambiar a contexto remoto
kubectl config use-context <nombre-contexto-remoto>

# Verificar acceso
kubectl cluster-info
```

## Configuración del Cluster

### Crear namespace dedicado:
```bash
kubectl create namespace kanban
kubectl config set-context --current --namespace=kanban
```

### Verificar namespace activo:
```bash
kubectl config view --minify | grep namespace
```

## Preparación de Imágenes

### Opción A: Usar imágenes del registry (recomendado)

Las imágenes se descargarán automáticamente desde Docker Hub cuando apliques los manifiestos.

### Opción B: Cargar imágenes locales (Kind)

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
kind load docker-image kanban-app:latest --name kanban-cluster
kind load docker-image kanban-ui:latest --name kanban-cluster
```

### Opción C: Cargar imágenes locales (Minikube)

```bash
# Configurar docker para usar minikube
eval $(minikube docker-env)

# Construir imágenes directamente en minikube
docker build -t kanban-app:latest ./kanban-app
docker build -t kanban-ui:latest ./kanban-ui
```

## Despliegue Completo

### 1. Clonar repositorio con submódulos (si aplica)

```bash
git clone --recurse-submodules https://github.com/braybaut/kanban-board.git
cd kanban-board/k8s-manifests  # o la carpeta donde estén los manifiestos
```

### 2. Desplegar PostgreSQL primero

```bash
kubectl apply -f postgres-deployment.yaml -n kanban
```

**¿Por qué primero?** La base de datos debe estar lista antes que el backend intente conectarse.

### 3. Esperar que PostgreSQL esté listo

```bash
kubectl wait --for=condition=ready pod -l app=kanban-postgres -n kanban --timeout=300s
```

### 4. Desplegar Backend

```bash
kubectl apply -f backend-deployment.yaml -n kanban
```

### 5. Desplegar Frontend

```bash
kubectl apply -f frontend-deployment.yaml -n kanban
```

### 6. Aplicar todos los manifiestos de una vez (alternativa)

```bash
kubectl apply -f . -n kanban
```

## Verificación y Acceso

### Verificar que todos los pods estén corriendo:

```bash
kubectl get pods -n kanban
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
kubectl get svc -n kanban
```

### Exponer la aplicación:

```bash
# Exponer frontend (puerto 8080 local → puerto 80 del servicio)
kubectl port-forward svc/kanban-ui 8080:80 -n kanban

# En otra terminal, exponer backend para debugging (opcional)
kubectl port-forward svc/kanban-app 8081:8080 -n kanban
```

### Acceder a la aplicación:

Abrir navegador en: **http://localhost:8080**

## Comprobaciones Rápidas

```bash
# Ver todos los recursos
kubectl get all -n kanban

# Ver pods con más detalle
kubectl get pods -n kanban -o wide

# Ver logs del backend
kubectl logs -l app=kanban-app -n kanban

# Ver logs del frontend
kubectl logs -l app=kanban-ui -n kanban

# Ver logs de PostgreSQL
kubectl logs -l app=kanban-postgres -n kanban

# Describir un pod problemático
kubectl describe pod <nombre-pod> -n kanban

# Probar conectividad interna
kubectl exec -it deployment/kanban-app-deployment -n kanban -- nc -zv kanban-postgres 5432
```

## Troubleshooting

### 1. **Pod en estado `ImagePullBackOff`**

**Problema:** No puede descargar la imagen Docker.

**Solución:**
```bash
# Verificar el error específico
kubectl describe pod <nombre-pod> -n kanban

# Si usas imágenes locales, asegúrate de cargarlas:
kind load docker-image kanban-app:latest --name kanban-cluster
```

### 2. **Backend no puede conectar a PostgreSQL**

**Problema:** Error de conexión a base de datos.

**Solución:**
```bash
# Verificar que PostgreSQL esté corriendo
kubectl get pods -l app=kanban-postgres -n kanban

# Verificar logs del backend
kubectl logs -l app=kanban-app -n kanban

# Probar conectividad
kubectl exec -it deployment/kanban-app-deployment -n kanban -- nc -zv kanban-postgres 5432
```

### 3. **Frontend muestra error de API**

**Problema:** No puede conectar con el backend.

**Solución:**
```bash
# Verificar que el backend esté corriendo
kubectl get pods -l app=kanban-app -n kanban

# Verificar variable de entorno API_URL
kubectl describe deployment kanban-ui-deployment -n kanban

# Probar backend directamente
kubectl port-forward svc/kanban-app 8081:8080 -n kanban
curl http://localhost:8081/api/health
```

### 4. **Port-forward no funciona**

**Problema:** No puedes acceder a localhost:8080.

**Solución:**
```bash
# Verificar que el servicio existe
kubectl get svc kanban-ui -n kanban

# Usar puerto diferente
kubectl port-forward svc/kanban-ui 9090:80 -n kanban

# Verificar que no hay otro proceso usando el puerto
lsof -i :8080
```

### 5. **Pods en estado `CrashLoopBackOff`**

**Problema:** El contenedor se reinicia constantemente.

**Solución:**
```bash
# Ver logs del pod
kubectl logs <nombre-pod> -n kanban --previous

# Ver eventos del namespace
kubectl get events -n kanban --sort-by='.lastTimestamp'

# Verificar recursos disponibles
kubectl top nodes
kubectl top pods -n kanban
```

### 6. **Namespace no encontrado**

**Problema:** Error "namespace kanban not found".

**Solución:**
```bash
# Crear el namespace
kubectl create namespace kanban

# Verificar namespaces existentes
kubectl get namespaces

# Configurar namespace por defecto
kubectl config set-context --current --namespace=kanban
```

## Limpieza

### Eliminar la aplicación:

```bash
# Eliminar todos los recursos del namespace
kubectl delete namespace kanban
```

### Eliminar cluster local:

```bash
# Kind
kind delete cluster --name kanban-cluster

# Minikube
minikube delete
```

## Referencias

- **Repositorio original:** https://github.com/wkrzywiec/kanban-board
- **Fork utilizado:** https://github.com/braybaut/kanban-board
- **Documentación Kubernetes:** https://kubernetes.io/docs/
- **Kind:** https://kind.sigs.k8s.io/
- **Minikube:** https://minikube.sigs.k8s.io/

---

**¿Problemas?** Revisa la sección [Troubleshooting](#troubleshooting) o abre un issue en el repositorio.
