# Laboratorio 6: Multiplataforma en AKS

## Juan Alejandro Sierra - U00178520
## Maria Herrera Taborda - U00173188

WordPress y PrestaShop en un mismo cluster Azure Kubernetes Service (AKS), cada uno con su propia base de datos MySQL, y Grafana propio para observar los pods de ambos portales.

## Arquitectura

| Namespace | Componente | Tipo de Service | Almacenamiento |
|---|---|---|---|
| wordpress | wordpress | LoadBalancer (publico) | PVC wordpress-pvc |
| wordpress | wordpress-db (MySQL 8.0) | ClusterIP (interno) | PVC wordpress-db-pvc |
| prestashop | prestashop | LoadBalancer (publico) | PVC prestashop-pvc |
| prestashop | prestashop-db (MySQL 8.0) | ClusterIP (interno) | PVC prestashop-db-pvc |
| monitoring | grafana, prometheus, kube-state-metrics | ClusterIP (interno) | Prometheus con emptyDir (24 h) |

Cada portal tiene su base de datos, su Secret y sus discos. Ninguna base de datos se expone a Internet. Grafana se accede con `kubectl port-forward`.

## Cluster usado

- Region: chilecentral
- Kubernetes: 1.36.5
- Nodos: 2 x Standard_DS2_v2, nivel Free
- Creado desde el portal de Azure (el template aks-app-template fallaba con la politica de regiones de Azure for Students y APIs retiradas).

## Despliegue

Requisitos: Azure CLI y kubectl, con sesion iniciada (`az login`).

```powershell
az aks get-credentials --resource-group rg-lab6-aks --name aks-lab6
```

### 1. WordPress

```powershell
kubectl apply -f k8s/wordpress/00-namespace.yaml
kubectl create secret generic wordpress-db-secret -n wordpress --from-literal=MYSQL_ROOT_PASSWORD='<clave-root>' --from-literal=MYSQL_DATABASE='wordpress' --from-literal=MYSQL_USER='wpuser' --from-literal=MYSQL_PASSWORD='<clave-wp>'
kubectl apply -f k8s/wordpress/db.yaml
kubectl apply -f k8s/wordpress/app.yaml
kubectl get svc wordpress -n wordpress
```

Abrir `http://<EXTERNAL-IP>` y completar el asistente.

### 2. PrestaShop

```powershell
kubectl apply -f k8s/prestashop/00-namespace.yaml
kubectl create secret generic prestashop-db-secret -n prestashop --from-literal=MYSQL_ROOT_PASSWORD='<clave-root>' --from-literal=MYSQL_DATABASE='prestashop' --from-literal=MYSQL_USER='psuser' --from-literal=MYSQL_PASSWORD='<clave-ps>'
kubectl apply -f k8s/prestashop/db.yaml
kubectl apply -f k8s/prestashop/app.yaml
kubectl get svc prestashop -n prestashop
```

En el asistente: servidor de BD `prestashop-db`, base `prestashop`, usuario `psuser`.

### 3. Grafana

```powershell
kubectl apply -f k8s/monitoring/00-namespace.yaml
kubectl create secret generic grafana-secret -n monitoring --from-literal=GF_ADMIN_USER='admin' --from-literal=GF_ADMIN_PASSWORD='<clave-grafana>'
kubectl apply -f k8s/monitoring/kube-state-metrics.yaml
kubectl apply -f k8s/monitoring/prometheus.yaml
kubectl apply -f k8s/monitoring/grafana.yaml
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Abrir `http://localhost:3000`, entrar con `admin` e importar `grafana/dashboard-lab6.json` (Dashboards, New, Import). Los archivos `*.example.yaml` son solo plantillas; las credenciales reales se crean con `kubectl create secret` y no estan en el repositorio.

## Prueba de persistencia

```powershell
kubectl rollout restart deployment/wordpress -n wordpress
kubectl rollout restart deployment/wordpress-db -n wordpress
kubectl rollout restart deployment/prestashop -n prestashop
kubectl rollout restart deployment/prestashop-db -n prestashop
kubectl get pods -A -w
```

Tras el reinicio, la pagina de WordPress y el producto de PrestaShop siguen presentes. Ver `evidencias/`.

## Apagado

- Pausa entre sesiones: `az aks stop -g rg-lab6-aks -n aks-lab6`
- Cierre del laboratorio: `az group delete -n rg-lab6-aks --yes`