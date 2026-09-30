# metrics-server en Talos

Métricas de CPU y memoria de nodos y pods para `kubectl top` y para los
`HorizontalPodAutoscaler`.

Referencia: <https://docs.siderolabs.com/kubernetes-guides/monitoring-and-observability/deploy-metrics-server>

| | |
|---|---|
| Chart | `metrics-server/metrics-server` 3.14.0 (metrics-server 0.9.0) |
| Aprobador de CSR | `kubelet-serving-cert-approver` v0.12.1 |
| TLS al kubelet | Verificado (sin `--kubelet-insecure-tls`) |

## Archivos

| Archivo | Contenido |
|---|---|
| `values.yaml` | Values de Helm de metrics-server |
| `../talos/kubelet-serving-certs.yaml` | Patch de Talos que activa `serverTLSBootstrap` en el kubelet |

Todos los comandos se corren desde la raíz del repo.

## 1. Aprobador de certificados del kubelet

Con `serverTLSBootstrap` el kubelet pide su certificado de serving con un CSR que
Kubernetes no aprueba solo. `kubelet-serving-cert-approver` los aprueba automáticamente
(solo los de tipo `kubernetes.io/kubelet-serving` que vienen de nodos del cluster).

Se instala antes del patch para que los CSR se aprueben apenas aparezcan:

```bash
kubectl apply -f https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/v0.12.1/deploy/standalone-install.yaml
kubectl -n kubelet-serving-cert-approver rollout status deploy/kubelet-serving-cert-approver
```

## 2. Patch de Talos en todos los nodos

```bash
talosctl -n $CONTROL_PLANE_IP,$WORKER1_IP,$WORKER2_IP patch machineconfig \
  --patch @talos/kubelet-serving-certs.yaml
```

Talos reinicia el kubelet (sin reboot). Verificar que cada nodo tenga un CSR aprobado:

```bash
kubectl get csr --sort-by=.metadata.creationTimestamp | grep kubelet-serving
```

Todos deben estar en `Approved,Issued`. Si quedan en `Pending`, revisar los logs del aprobador:

```bash
kubectl -n kubelet-serving-cert-approver logs deploy/kubelet-serving-cert-approver
```

## 3. Instalar metrics-server

```bash
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/ && helm repo update
helm install metrics-server metrics-server/metrics-server -n kube-system --version 3.14.0 \
  -f metrics-server/values.yaml
```

Para cambiar la config después:

```bash
helm upgrade metrics-server metrics-server/metrics-server -n kube-system --version 3.14.0 \
  -f metrics-server/values.yaml
```

## 4. Verificar

```bash
kubectl get apiservice v1beta1.metrics.k8s.io   # AVAILABLE=True
kubectl top nodes
kubectl top pods -A
```

La primera lectura tarda ~30 s después de que el pod está `Ready`.

Si `kubectl top` falla, mirar los logs de metrics-server:

```bash
kubectl -n kube-system logs deploy/metrics-server
```

- `x509: cannot validate certificate ... doesn't contain any IP SANs`: el kubelet sigue con el
  certificado autofirmado. Revisar el paso 2 (patch aplicado y CSR aprobados).

Después de un `helm upgrade`, confirmar que quedó un solo pod. Si el nuevo no pasa el readiness,
el viejo sigue sirviendo y esconde el problema:

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=metrics-server
```
