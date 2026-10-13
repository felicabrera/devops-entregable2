# Entregable 1 DevOps: Despliegue Blue/Green

Notes API desplegada en Kubernetes (Docker Desktop) usando una estrategia
Blue/Green: v1 (blue) y v2 (green) corren en paralelo, y el tráfico se
cambia entre ellas parcheando el selector del Service, sin downtime.

## Prerrequisitos

- Docker Desktop con Kubernetes habilitado (Settings > Kubernetes > Enable Kubernetes)
- `kubectl` (incluido con Docker Desktop)

Verificar que el cluster esté activo y apuntando a Docker Desktop:

```bash
kubectl config use-context docker-desktop
kubectl get nodes
```

## Diferencia entre v1 y v2

v2 agrega un endpoint nuevo, `GET /stats`, que devuelve la cantidad total
de notas guardadas. Este endpoint solo se registra cuando `APP_VERSION`
empieza con `v2` (ver `main.py`), así que blue (v1) no lo tiene y green
(v2) sí. `/` y `/health` siguen reportando `version` y `color` en ambas.

Para verificarlo una vez desplegado (pasos 2 a 4 más abajo):

```bash
curl -s -o /dev/null -w 'blue /stats -> %{http_code}\n' http://localhost/stats
# blue /stats -> 404

./switch-to-green.sh
curl http://localhost/stats
# {"total_notes":0,"version":"v2.0.0"}
```

## 1. Construir las imágenes

El Kubernetes de Docker Desktop comparte el mismo daemon de Docker, así que
las imágenes construidas localmente ya son visibles para el cluster (no
hace falta cargarlas aparte).

```bash
docker build -t devops-app:v1 .
docker build -t devops-app:v2 .
```

## 2. Desplegar Blue y Green

```bash
kubectl apply -f k8s/blue-deployment.yaml
kubectl apply -f k8s/green-deployment.yaml
kubectl apply -f k8s/service.yaml
```

El Service arranca apuntando a `version: blue`.

```bash
kubectl get pods -l app=devops-app -L version
```

## 3. Acceder al servicio

`k8s/service.yaml` es un Service de tipo `LoadBalancer`. El Kubernetes de
Docker Desktop expone los Services `LoadBalancer` directamente en
`localhost`, en el `port` del Service (no hace falta `minikube service` ni
un NodePort):

```bash
curl http://localhost/health
# {"status":"ok","health_param":"default","version":"v1.0.0","color":"blue"}
```

## 4. Cutover a Green

```bash
./switch-to-green.sh
curl http://localhost/health
# {"status":"ok","health_param":"default","version":"v2.0.0","color":"green"}
```

Para volver atrás: `./switch-to-blue.sh`.

## 5. Verificar el cambio sin downtime

```bash
while true; do curl -s http://localhost/health; echo; sleep 1; done
```

Mientras el loop corre, ejecutar `./switch-to-green.sh` en otra terminal;
la salida pasa de blue a green sin ningún request fallido.

## Troubleshooting: imagen desactualizada

Si reconstruís `devops-app:v1` o `devops-app:v2` después de ya haberlos
desplegado, Kubernetes puede seguir usando la imagen vieja cacheada bajo
ese mismo tag (`imagePullPolicy: IfNotPresent` no detecta que el contenido
cambió). Si ves código viejo corriendo en el pod, forzá la limpieza del
cache de imágenes del nodo:

```bash
kubectl debug node/desktop-control-plane -it --image=alpine --profile=sysadmin -- \
  chroot /host crictl rmi docker.io/library/devops-app:v1 docker.io/library/devops-app:v2
kubectl delete pod node-debugger-desktop-control-plane-*
kubectl rollout restart deployment/devops-app-blue deployment/devops-app-green
```

# Entregable 2: Monitoreo y alertas

La app expone métricas Prometheus en `GET /metrics`:

| Métrica | Tipo | Descripción |
|---|---|---|
| `notes_created_total` | Counter | Notas creadas desde que arrancó el proceso |
| `notes_total` | Gauge | Cantidad de notas guardadas actualmente |
| `http_request_duration_seconds` | Histogram | Llamadas por `method`, `endpoint` y `status` |

La alerta `MuchasNotasCreadas` (`monitoring/alerts.yml`) se dispara cuando
se crean más de 10 notas (X) en una ventana de 5 minutos (Y):

```
sum(increase(notes_created_total[5m])) > 10
```

Para cambiar X o Y, editar esa expresión.

## Opción A: Docker Compose

```bash
docker compose up -d --build
```

| Servicio | URL |
|---|---|
| App | http://localhost:8000 |
| Prometheus | http://localhost:9091 (Status > Targets) |
| Grafana | http://localhost:3001 (admin / admin), dashboard "Notes API" |

Prometheus toma el target `app:8000/metrics` de `monitoring/prometheus.yml`.
Grafana provisiona el datasource y el dashboard desde `monitoring/grafana/`.

Para disparar la alerta:

```bash
for i in $(seq 1 12); do curl -s -X POST http://localhost:8000/add/nota$i > /dev/null; done
```

Y verla en http://localhost:9091/alerts (estado `firing`).

## Opción B: Kubernetes

Se despliega junto a la app de Blue/Green. Hay que reconstruir las imágenes
(para que incluyan `/metrics`) y aplicar los manifiestos:

```bash
docker build -t devops-app:v1 .
docker build -t devops-app:v2 .
kubectl apply -f k8s/blue-deployment.yaml -f k8s/green-deployment.yaml -f k8s/service.yaml
./deploy-monitoring.sh
```

Prometheus descubre cada pod de la app (blue y green) por su label
`app=devops-app`, así que cada réplica es un target propio.

- Prometheus: http://localhost:9091
- Grafana: http://localhost:3001 (admin / admin)

Los datos de Prometheus viven en el PersistentVolumeClaim `prometheus-data`
(2Gi), por lo que sobreviven a reinicios del pod. En Compose se usa el volumen
nombrado `prometheus-data` (se borra con `docker compose down -v`).

No levantar Compose y Kubernetes a la vez, usan los mismos puertos.
