# Grafana en Talos

Primera app stateful del lab: prueba toda la pila en un solo despliegue.

```
Navegador ──https──▶ 192.168.68.200 (MetalLB) ──▶ Traefik ──▶ Service ──▶ Pod Grafana
                                                        │
                     Secret TLS ◀── cert-manager (lab-ca) │
                                                        └──▶ PVC en Longhorn
```

| | |
|---|---|
| Chart | `grafana/grafana` 10.5.15 (Grafana 12.3.1) |
| Host | `grafana.192.168.68.200.nip.io` |
| PVC | `grafana` 5Gi, StorageClass `longhorn` |
| Estrategia | `Recreate` (1 réplica + PVC RWO, ver abajo) |
| Password inicial | `admin` / `admin` (fuerza cambio al primer login) |

Los dashboards y datasources quedan guardados en el PVC, así que persisten
entre reinicios del pod: es lo que hace que la app sirva de prueba real de
Longhorn.

Todos los comandos se corren desde la raíz del repo.

## 1. Instalar

```bash
helm repo add grafana https://grafana.github.io/helm-charts && helm repo update
helm install grafana grafana/grafana \
  -n grafana --create-namespace --version 10.5.15 \
  -f grafana/values.yaml
```

Esperar:

```bash
kubectl -n grafana rollout status deploy/grafana
kubectl -n grafana get pvc        # grafana BOUND
kubectl -n grafana get certificate grafana-ingress-tls   # READY=True
```

## 2. Ingress

Archivo `grafana/ingress.yaml` (recurso aparte del chart, igual patrón que
`ingress/test-whoami.yaml`).

```bash
kubectl apply -f grafana/ingress.yaml
```

```bash
kubectl -n grafana get certificate grafana-ingress-tls   # READY=True
```

## 3. Usar

```bash
kubectl -n grafana get secret grafana -o jsonpath='{.data.admin-user}' | base64 -d; echo
kubectl -n grafana get secret grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

- URL: <https://grafana.192.168.68.200.nip.io>
- Login: `admin` / `admin` (pide cambiarlo la primera vez)
- La CA `lab-ca` tiene que estar importada en el cliente (paso 1.3 de `ingress/INGRESS.md`).

Qué se probó con este despliegue:

| Capa | Cómo se valida |
|---|---|
| MetalLB | Traefik responde en `192.168.68.200` |
| Traefik | El Ingress rutea al Service `grafana` |
| cert-manager | `grafana-ingress-tls` emitido por `lab-ca` |
| Longhorn | `grafana` BOUND; el login sigue válido tras `kubectl -n grafana rollout restart deploy/grafana` |

## 5. Por qué `Recreate`

Con 1 réplica y un PVC `ReadWriteOnce`, la estrategia por defecto (`RollingUpdate`)
puede dead-lockear un restart: el pod nuevo puede caer en otro nodo mientras el
volumen sigue adjunto al viejo (`Multi-Attach error`), y como
`maxUnavailable=0` el viejo nunca baja. `Recreate` baja el pod viejo antes de
crear el nuevo. Lo mismo aplica a cualquier app single-replica con PVC RWO.

También se deshabilita el init container `init-chown-data` del chart: corre como
root (violaba PodSecurity `restricted`); los permisos los resuelve `fsGroup`.

## 4. Datos de prueba (opcional)

El chart trae un datasource de ejemplo apagado; para ver algo enseguida,
descomentar la sección `datasources` en `values.yaml` para apuntar a un
prometheus si se despliega después, o cargar un dashboard de la comunidad
(Get dashboards → ID 15760, "Kubernetes cluster monitoring").

## Problemas comunes

| Síntoma | Causa probable |
|---|---|
| PVC `Pending` | Longhorn no instalado, o el nodo no tiene el disco de datos etiquetado |
| `READY=False` el certificado | `kubectl -n grafana describe certificate grafana-ingress-tls` |
| 404 de Traefik | Revisar `host` del Ingress vs URL |
| Pod viejo corriendo y nuevo en `Init:0/1` con `Multi-Attach error` | Rollout con `RollingUpdate` y PVC RWO: usar estrategia `Recreate` (sección 5) |
| No recuerda el usuario nuevo tras reiniciar | El PVC no está montado: `kubectl -n grafana describe pod -l app.kubernetes.io/name=grafana` |