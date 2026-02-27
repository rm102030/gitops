# Guía: Colima + ArgoCD + mi-app en macOS

Esta guía te llevará desde la instalación de Colima hasta desplegar la aplicación mi-app con ArgoCD.

## Requisitos Previos

- macOS
- Homebrew instalado

## 1. Instalar Colima

```bash
# Instalar Colima y Docker CLI
brew install colima docker kubectl

# Iniciar Colima con configuración básica
colima start --cpu 4 --memory 8 --disk 50

# Verificar que está corriendo
colima status
kubectl cluster-info
```

## 2. Instalar ArgoCD

```bash
# Crear namespace para ArgoCD
kubectl create namespace argocd

# Instalar ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Esperar a que todos los pods estén listos
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s

# Obtener la contraseña inicial del admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo ""

# Hacer port-forward para acceder a la UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Accede a ArgoCD en: https://localhost:8080
- Usuario: `admin`
- Contraseña: (la que obtuviste en el comando anterior)

## 3. Clonar el Repositorio GitOps

```bash
# Clonar el repositorio
cd ~/Documents/argocd
git clone https://github.com/rm102030/gitops.git
cd gitops
```

## 4. Desplegar mi-app con ArgoCD

### Opción A: Usando kubectl

```bash
# Aplicar la configuración de la Application
kubectl apply -f argocd/mi-app-application.yaml

# Verificar el estado
kubectl get applications -n argocd
```

### Opción B: Usando ArgoCD CLI

```bash
# Instalar ArgoCD CLI
brew install argocd

# Login
argocd login localhost:8080 --insecure

# Crear la aplicación
argocd app create mi-app \
  --repo https://github.com/rm102030/gitops.git \
  --path mi-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --auto-prune \
  --self-heal \
  --sync-option CreateNamespace=true

# Sincronizar (si no está en modo automático)
argocd app sync mi-app
```

## 5. Configurar Ingress (Opcional)

Si quieres acceder a mi-app vía http://mi-app.local:

```bash
# Instalar nginx-ingress
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

# Agregar entrada al /etc/hosts
echo "127.0.0.1 mi-app.local" | sudo tee -a /etc/hosts

# Hacer port-forward del ingress
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 80:80
```

Accede a: http://mi-app.local

## 6. Verificar el Despliegue

```bash
# Ver el estado de la aplicación en ArgoCD
kubectl get application mi-app -n argocd

# Ver los recursos desplegados
kubectl get all -n default -l app=mi-app

# Ver logs del pod
kubectl logs -n default -l app=mi-app -f
```

## Recursos Desplegados

La aplicación mi-app incluye:
- **ConfigMap**: mi-app-html (contenido HTML)
- **Deployment**: mi-app (nginx:1.25-alpine)
- **Service**: mi-app (ClusterIP)
- **Ingress**: mi-app (http://mi-app.local/)

## Comandos Útiles

```bash
# Ver estado de ArgoCD
kubectl get pods -n argocd

# Ver aplicaciones
kubectl get applications -n argocd

# Ver detalles de mi-app
kubectl describe application mi-app -n argocd

# Forzar sincronización
kubectl patch application mi-app -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"revision":"HEAD"}}}'

# Detener Colima
colima stop

# Reiniciar Colima
colima start
```

## Troubleshooting

### ArgoCD no sincroniza automáticamente
```bash
# Verificar la configuración de syncPolicy
kubectl get application mi-app -n argocd -o yaml | grep -A 5 syncPolicy
```

### Pods no inician
```bash
# Ver eventos
kubectl get events -n default --sort-by='.lastTimestamp'

# Ver logs del pod
kubectl logs -n default <pod-name>
```

### No puedo acceder a ArgoCD UI
```bash
# Verificar que el port-forward está activo
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Actualizar la Aplicación

1. Modifica los archivos en `mi-app/`
2. Haz commit y push al repositorio
3. ArgoCD sincronizará automáticamente (si está configurado)

```bash
cd ~/Documents/argocd/gitops
# Hacer cambios...
git add .
git commit -m "Actualizar mi-app"
git push origin main
```

ArgoCD detectará los cambios y aplicará la actualización automáticamente.
