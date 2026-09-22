# Monitoreo del cluster con Prometheus (kube-prometheus-stack)

Métricas del cluster (API server, etcd, kubelet, kube-proxy, controller-manager,
scheduler, coredns, node-exporter) con reglas y alertas listas, más datasource
de Prometheus en la Grafana ya existente.

| | |
|---|---|
| Chart | `prometheus-community/kube-prometheus-stack` 91.4.1 |
| Namespace | `monitoring` (pod security `privileged`: node-exporter lo necesita en Talos) |
| Prometheus | PVC 10Gi en Longhorn, retención 7 días |
| Alertmanager | PVC 2Gi en Longhorn |
| Grafana | La ya desplegada en `grafana/` (la del chart va deshabilitada) |
| Hosts | `prometheus.192.168.68.200.nip.io`, `alertmanager.192.168.68.200.nip.io` |

Flujo de datos:

```
kubelet/cAdvisor ─┐
API server ────────┤
etcd ──────────────┼──▶ Prometheus ──▶ Grafana (datasource)
coredns ───────────┤        │
node-exporter ─────┘        ▼
                        Alertmanager
```

Todos los comandos se corren desde la raíz del repo.

## 1. Namespace

`monitoring/namespace.yaml` lleva pod security `privileged` (igual que
`metallb-system`): el node-exporter usa `hostNetwork` y mounts de `/proc` y
`/sys`.

```bash
kubectl apply -f monitoring/namespace.yaml
```

## 2. Métricas de los componentes del control plane (patch de Talos)

Por defecto Talos bindea las métricas de kube-proxy, kube-scheduler y
kube-controller-manager a `127.0.0.1` (hardening): Prometheus scrapea la IP
del nodo y la conexión es rechazada. El patch `talos/controlplane-metrics.yaml`
las expone en `0.0.0.0`. **Se aplica a todos los nodos** (kube-proxy corre en
todos):

```bash
export TALOSCONFIG="$PWD/_out/talosconfig"
talosctl -n 192.168.68.83 patch machineconfig --patch @talos/controlplane-metrics.yaml
talosctl -n 192.168.68.82,192.168.68.84 patch machineconfig --patch @talos/controlplane-metrics.yaml
```

Los flags de scheduler/controller-manager hacen efecto sin reboot. kube-proxy
es un DaemonSet aplicado en el bootstrap por Talos y el patch **no** re-crea
el que ya existe (Talos lo marca como aplicado en el ConfigMap
`talos-bootstrap-manifests-inventory` de `kube-system` y no lo toca más).
Re-crearlo:

```bash
# Sacar el ownership annotation y borrarlo
kubectl -n kube-system patch ds kube-proxy --type=json \
  -p '[{"op":"remove","path":"/metadata/annotations/config.k8s.io~1owning-inventory"}]'
kubectl -n kube-system delete ds kube-proxy

# Sacarlo del inventory y aplicar el manifest renderizado por Talos
# (ya trae --metrics-bind-address=0.0.0.0:10249)
kubectl -n kube-system get configmap talos-bootstrap-manifests-inventory -o json | \
  python3 -c 'import sys,json; d=json.load(sys.stdin); [d["data"].pop(k) for k in list(d["data"]) if "proxy" in k.lower()]; print(json.dumps(d))' | \
  kubectl apply -f -
talosctl -n 192.168.68.83 get manifests 10-kube-proxy -o yaml > /tmp/kp.yaml
python3 -c 'import yaml; m=yaml.safe_load(open("/tmp/kp.yaml")); print("\n---\n".join(yaml.safe_dump(o,sort_keys=False) for o in m["spec"]))' | kubectl apply -f -
kubectl -n kube-system rollout status ds/kube-proxy
```

Verificar que escucha en todas las interfaces:

```bash
talosctl -n 192.168.68.82,192.168.68.83,192.168.68.84 netstat -l | grep 10249
# :::10249 en los tres nodos
```

## 3. Instalar el stack

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts && helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --version 91.4.1 \
  -f monitoring/values.yaml
```

```bash
kubectl -n monitoring rollout status deploy/monitoring-kube-prometheus-operator
kubectl -n monitoring get pods
```

## 4. Ingress

```bash
kubectl apply -f monitoring/ingress.yaml
```

```bash
kubectl -n monitoring get certificate   # los dos en READY=True
```

- Prometheus: <https://prometheus.192.168.68.200.nip.io>
- Alertmanager: <https://alertmanager.192.168.68.200.nip.io>

Targets (deben ser 26/26 up):

```bash
curl --cacert lab-ca.crt -s https://prometheus.192.168.68.200.nip.io/api/v1/targets | \
  python3 -c 'import sys,json; t=json.load(sys.stdin)["data"]["activeTargets"]; print(sum(1 for x in t if x["health"]=="up"), "/", len(t), "up")'
```

## 5. Conectar con Grafana

En Grafana: Connections → Data sources → Add data source → **Prometheus**

- URL: `http://monitoring-kube-prometheus-prometheus.monitoring.svc:9090`
- Save & Test

O via API:

```bash
curl -u admin:<password> https://grafana.192.168.68.200.nip.io/api/datasources \
  -H 'Content-Type: application/json' -k -d '{
    "name": "Prometheus", "type": "prometheus", "isDefault": true,
    "url": "http://monitoring-kube-prometheus-prometheus.monitoring.svc:9090",
    "access": "proxy"}'
```

Si el login `admin` falla con el password del Secret (el usuario se cambió vía
UI en algún momento), resetearlo desde dentro del pod:

```bash
kubectl -n grafana exec deploy/grafana -- grafana-cli --homepath /usr/share/grafana admin reset-admin-password admin
```

Después, Dashboards → New → Import. IDs útiles:

| ID | Dashboard |
|---|---|
| 15757 | Kubernetes / Views / Nodes |
| 15758 | Kubernetes / Views / Pods |
| 15759 | Kubernetes / Views / Global |
| 1860 | Node Exporter Full |
| 747 | kubelet |

## 6. Longhorn en Grafana (opcional)

Longhorn expone métricas en el Service `longhorn-metrics` de `longhorn-system`.
Para scrapearlas: un `ServiceMonitor` con label `release: monitoring` (el
selector del stack); el dashboard de Longhorn (ID 13032) las usa.

## Problemas comunes

| Síntoma | Causa probable |
|---|---|
| Pods de node-exporter `CreateContainerConfigError` | Falta el namespace con pod security `privileged` (paso 1) |
| Targets de scheduler/controller-manager/kube-proxy `down`, `connection refused` | Métricas en `127.0.0.1` por hardening de Talos: falta el paso 2 (patch + re-crear kube-proxy) |
| PVC `Pending` | Longhorn sin disco en ese nodo, o StorageClass mal escrito |
| Datasource en Grafana sin datos | URL sin el namespace completo: tiene que ser `http://<svc>.<ns>.svc:9090` |
| 401 con `admin:admin` en Grafana | Password cambiado via UI: resetearlo con `grafana-cli` (paso 5) |