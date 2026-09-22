# Dashboards de Grafana como código (sidecar)

Los JSON de `monitoring/dashboards/` son la fuente de verdad: el sidecar de
Grafana los sincroniza automáticamente (provisioning), así que importarlos a
mano en la UI no es necesario y los cambios manuales se pisan al reiniciar.

| Archivo | Dashboard | Origen |
|---|---|---|
| `15757.json` | Kubernetes / Views / Global | grafana.com ID 15757 |
| `15758.json` | Kubernetes / Views / Namespaces | grafana.com ID 15758 |
| `15759.json` | Kubernetes / Views / Nodes | grafana.com ID 15759 |
| `1860.json` | Node Exporter Full | grafana.com ID 1860 |
| `747.json` | Kubernetes Pod Metrics | grafana.com ID 747 |

Fuente original: <https://grafana.com/grafana/dashboards/> (descargar con
`<id>/revisions/<n>/download`; el `revision` es el número público, no el
`items[].id` interno).

## Cómo funciona

1. `grafana/values.yaml` habilita el sidecar `grafana-dashboard-scaper`
   (`grafana.sidecar.dashboards.enabled: true`).
2. El sidecar lee los ConfigMaps con label
   `grafana_dashboard: "1"` del namespace `grafana` y los escribe en
   `/tmp/dashboards` dentro del pod.
3. El provisioning de Grafana (incluido en el chart) levanta lo que caiga en
   ese directorio.

## Agregar o cambiar un dashboard

1. Poner el JSON en `monitoring/dashboards/`.
2. Agregar el archivo a `configMapGenerator` en `kustomization.yaml`.
3. Aplicar. **Ojo:** `kubectl apply` falla con dashboards grandes (1860 pesa
   ~670KB y supera el límite de 256KB del ConfigMap por la annotation de
   tracking de apply), así que el flujo es create + label:

```bash
# ConfigMaps chicos: kustomize normal
kubectl apply -k monitoring/dashboards

# Dashboards grandes (como 1860): create + label
kubectl -n grafana create cm dashboard-1860 --from-file=1860.json=monitoring/dashboards/1860.json \
  --dry-run=client -o yaml | sed 's/^  labels: null/  labels: {}/' | kubectl create -f - 2>/dev/null || \
kubectl -n grafana create cm dashboard-1860 --from-file=1860.json=monitoring/dashboards/1860.json
kubectl -n grafana label cm dashboard-1860 grafana_dashboard=1 --overwrite
```

Para actualizar un JSON grande ya existente: borrar el ConfigMap y repetir el
create (apply no se puede usar con estos).

## Verificar

```bash
kubectl -n grafana get cm -l grafana_dashboard=1
```

El sidecar sincroniza cada ~1 min; en Grafana, Dashboards → carpeta
"General" (o la que defina el ConfigMap).

## Notas

- Las variables de plantilla (`cluster`, `job`, `node`) se resuelven solas al
  abrir el dashboard.
- El datasource es el `Prometheus` creado en el paso 5 de `MONITORING.md`
  (UID autogenerado; los dashboards usan la variable `DS_PROMETHEUS` que
  Grafana resuelve al datasource default).
- Para versionar cambios propios de un JSON: editarlo acá y volver a aplicar
  el ConfigMap. No editarlo solo en la UI: se pierde en el próximo sync.