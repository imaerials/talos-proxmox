# MetalLB en Talos (modo L2)

Da IPs de la red local (`192.168.68.x`) a los servicios `type: LoadBalancer`.
En modo L2 un worker "responde" por cada IP vía ARP; no hace falta configurar nada en el router.

Referencia: <https://metallb.io/installation/>

| | |
|---|---|
| Chart | `metallb/metallb` 0.16.1 |
| Modo | L2 (sin BGP) |
| Pool | `192.168.68.200-192.168.68.220` (fuera del DHCP del router) |
| kube-proxy | `nftables` (no requiere `strictARP`, eso es solo para IPVS) |
| Nodos que anuncian | Solo workers: Talos etiqueta el control plane con `node.kubernetes.io/exclude-from-external-load-balancers` |

## Archivos

| Archivo | Contenido |
|---|---|
| `namespace.yaml` | Namespace `metallb-system` con pod security `privileged` (el `speaker` usa `hostNetwork` y `NET_RAW`) |
| `values.yaml` | Values de Helm: `frrk8s.enabled: false` (FRR-K8s solo se usa para BGP) |
| `ip-pool.yaml` | `IPAddressPool` `default-pool` + `L2Advertisement` `default-l2` |
| `test-lb.yaml` | nginx + `Service` `LoadBalancer` para probar |

Todos los comandos se corren desde la raíz del repo.

## 1. Namespace

```bash
kubectl apply -f metalLb/namespace.yaml
```

## 2. Instalar MetalLB

```bash
helm repo add metallb https://metallb.github.io/metallb && helm repo update
helm install metallb metallb/metallb -n metallb-system --version 0.16.1 \
  -f metalLb/values.yaml
```

Esperar a que esté listo (los CRDs y el webhook tienen que estar arriba antes del paso 3):

```bash
kubectl -n metallb-system rollout status deploy/metallb-controller
kubectl -n metallb-system rollout status ds/metallb-speaker
kubectl -n metallb-system get pods -o wide
```

## 3. Pool de IPs y anuncio L2

```bash
kubectl apply -f metalLb/ip-pool.yaml
kubectl -n metallb-system get ipaddresspools,l2advertisements
```

Si el `apply` falla con un error del webhook (`failed calling webhook ... connection refused`),
el controller todavía no terminó de arrancar: esperar unos segundos y repetir.

## 4. Probar

```bash
kubectl apply -f metalLb/test-lb.yaml
kubectl get svc lb-test -w     # EXTERNAL-IP debe tomar una IP del rango
```

Desde otra máquina de la red:

```bash
curl http://<EXTERNAL-IP>
```

Ver qué worker está anunciando la IP:

```bash
kubectl describe svc lb-test | grep -A5 Events   # "announcing from node ..."
```

Limpiar:

```bash
kubectl delete -f metalLb/test-lb.yaml
```

## 5. Uso

- Cualquier `Service` con `type: LoadBalancer` recibe una IP del pool automáticamente.
- Para fijar una IP concreta del pool:

  ```yaml
  metadata:
    annotations:
      metallb.io/loadBalancerIPs: 192.168.68.200
  ```

## Problemas comunes

| Síntoma | Causa probable |
|---|---|
| `EXTERNAL-IP` queda en `<pending>` | No hay `IPAddressPool`/`L2Advertisement`, o el pool se quedó sin IPs |
| La IP se asigna pero no responde | La IP está en el rango DHCP o la usa otro equipo (conflicto ARP), o el `speaker` no corre en los workers |
| Pods del `speaker` no arrancan | Falta el label `privileged` en el namespace (paso 1) |
