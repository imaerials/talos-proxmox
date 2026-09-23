# talos-proxmox

Cluster Kubernetes de laboratorio con Talos Linux sobre Proxmox: 1 control plane y 2 workers,
con almacenamiento Longhorn, IPs de LoadBalancer con MetalLB e ingress HTTPS con Traefik y
cert-manager.

| Nodo | IP | Rol |
|---|---|---|
| `talos-cp-01` | `192.168.68.83` | Control plane |
| `talos-w-01` | `192.168.68.82` | Worker (+ disco Longhorn) |
| `talos-w-02` | `192.168.68.84` | Worker (+ disco Longhorn) |

Diagrama del cluster: [`docs/talos-lab-infra.html`](docs/talos-lab-infra.html).

## Orden de instalación

Todo parte del cluster del paso 1, y Traefik (paso 4) necesita la IP que le da MetalLB (paso 3).
Todos los comandos se corren desde la raíz del repo.

| # | Doc | Qué hace |
|---|---|---|
| 1 | [`Setup-Final.md`](Setup-Final.md) | VMs en Proxmox, config de Talos, bootstrap y disco de datos |
| 2 | [`longhorn/LONGHORN.md`](longhorn/LONGHORN.md) | Almacenamiento persistente (StorageClass `longhorn` por defecto) |
| 3 | [`metalLb/METALLB.md`](metalLb/METALLB.md) | IPs `192.168.68.200-220` para Services `LoadBalancer` |
| 4 | [`ingress/INGRESS.md`](ingress/INGRESS.md) | Traefik en `192.168.68.200` y certificados de la CA del lab |

## Estructura

```
talos/      patches de machine config de Talos
longhorn/   values y namespace de Longhorn
metalLb/    values, namespace y pool de IPs de MetalLB
ingress/    values de Traefik y cert-manager, CA del lab
docs/       diagrama del cluster
_out/       configs generadas por talosctl (con secretos, ignorado por git)
```
