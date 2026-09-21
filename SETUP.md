# Talos + Longhorn en Proxmox — instalación desde cero

Guía para reconstruir el cluster `talos-proxmox-cluster` tal como está hoy:
Talos v1.14.0 · Kubernetes v1.36.3 · Flannel · Longhorn 1.12.1 como storage por defecto.

| Nodo | VM | IP | vCPU | RAM | Disco | Rol |
|---|---|---|---|---|---|---|
| `controlplane1` | 210 | 192.168.68.63 | 2 | 4 GB | 40 GB | control plane |
| `worker1` | 211 | 192.168.68.71 | 4 | 6 GB | 60 GB | worker + Longhorn |
| `worker2` | 212 | 192.168.68.74 | 4 | 6 GB | 60 GB | worker + Longhorn |

Referencias oficiales (revisadas 2026-09-21):
- Talos en Proxmox: <https://docs.siderolabs.com/talos/v1.14/platform-specific-installations/virtualized-platforms/proxmox>
- Longhorn en Talos: <https://longhorn.io/docs/latest/advanced-resources/os-distro-specific/talos-linux-support/>

> La versión más reciente de Talos es la **v1.14.1**. Esta guía fija la v1.14.0 para
> replicar exactamente el cluster actual; para usar otra, cambiá `TALOS_VERSION` y
> nada más.

---

## 0. Variables

Todo el documento usa estas variables. Se corre desde la raíz del repo.

```bash
cd ~/talos-proxmox

export TALOS_VERSION=v1.14.0
export CLUSTER_NAME=talos-proxmox-cluster
export CONTROL_PLANE_IP=192.168.68.63
export WORKER1_IP=192.168.68.71
export WORKER2_IP=192.168.68.74
export TALOSCONFIG="$PWD/_out/talosconfig"
```

Las IPs salen por DHCP: **reservalas en el router** (por MAC) para que no cambien
entre reinicios. El endpoint del cluster (`https://$CONTROL_PLANE_IP:6443`) queda
grabado en los certificados y en la config de cada nodo.

---

## 1. Herramientas en el cliente

```bash
# talosctl — conviene que coincida con la versión de Talos del cluster
curl -sL https://talos.dev/install | sh
talosctl version --client

# kubectl y helm (si no están)
kubectl version --client
helm version --short
```

> `talosctl gen config` genera la config con el contrato de versión del **cliente**.
> Si tu `talosctl` es más viejo que `TALOS_VERSION` (por ejemplo v1.13.x), actualizalo
> antes del paso 4.

---

## 2. Imagen de Talos con extensiones (Image Factory)

Talos es inmutable: lo que el sistema necesite tiene que venir dentro de la imagen.
Longhorn necesita `iscsi-tools` y `util-linux-tools`; `qemu-guest-agent` hace que
Proxmox vea la IP y pueda apagar la VM de forma ordenada.

`talos/schematic.yaml` (ya está en el repo):

```yaml
customization:
  systemExtensions:
    officialExtensions:
      - siderolabs/qemu-guest-agent
      - siderolabs/iscsi-tools
      - siderolabs/util-linux-tools
```

Obtener el ID del schematic:

```bash
curl -s -X POST --data-binary @talos/schematic.yaml https://factory.talos.dev/schematics
# {"id":"53513e54bb39202f35694412577a6bc53d484744d35a126e5d42ef34785c0d83", ...}

export SCHEMATIC=53513e54bb39202f35694412577a6bc53d484744d35a126e5d42ef34785c0d83
```

El ID es un hash del contenido: el mismo YAML devuelve siempre el mismo ID.
Con él salen las dos imágenes:

| Uso | URL |
|---|---|
| ISO de arranque (paso 3) | `https://factory.talos.dev/image/$SCHEMATIC/$TALOS_VERSION/metal-amd64.iso` |
| Installer (paso 4) | `factory.talos.dev/metal-installer/$SCHEMATIC:$TALOS_VERSION` |

> La ISO solo sirve para arrancar la primera vez. Lo que queda instalado en disco es el
> **installer** configurado en `machine.install.image`. Si ahí queda una imagen sin
> extensiones, Longhorn no va a funcionar aunque hayas arrancado desde la ISO correcta.

---

## 3. Crear las VMs — en el host Proxmox

Descargar la ISO:

```bash
cd /var/lib/vz/template/iso/
SCHEMATIC=53513e54bb39202f35694412577a6bc53d484744d35a126e5d42ef34785c0d83
TALOS_VERSION=v1.14.0
curl -L -o talos-${TALOS_VERSION}-longhorn-metal-amd64.iso \
  https://factory.talos.dev/image/${SCHEMATIC}/${TALOS_VERSION}/metal-amd64.iso
```

Crear las VMs:

```bash
STORAGE=local-lvm
ISO=local:iso/talos-v1.14.0-longhorn-metal-amd64.iso
BRIDGE=vmbr0

create_talos_vm() {
  local vmid=$1 name=$2 cores=$3 mem=$4 disk=$5
  qm create "$vmid" \
    --name "$name" \
    --ostype l26 \
    --machine q35 \
    --bios ovmf \
    --efidisk0 ${STORAGE}:1,efitype=4m,pre-enrolled-keys=0 \
    --cpu host \
    --cores "$cores" \
    --memory "$mem" \
    --balloon 0 \
    --scsihw virtio-scsi-pci \
    --scsi0 ${STORAGE}:${disk},cache=writethrough,discard=on,ssd=1 \
    --ide2 ${ISO},media=cdrom \
    --boot order='scsi0;ide2' \
    --net0 virtio,bridge=${BRIDGE} \
    --agent enabled=1 \
    --serial0 socket
}

create_talos_vm 210 talos-cp-01 2 4096 40
create_talos_vm 211 talos-w-01  4 6144 60
create_talos_vm 212 talos-w-02  4 6144 60

qm start 210 && qm start 211 && qm start 212
```

Por qué cada opción, según la doc oficial:

- **`q35`, `ovmf` y un EFI disk de 4 MB:** máquina moderna con UEFI.
- **`--cpu host`:** mejor rendimiento. En Proxmox anterior a la 8 hace falta `kvm64` con flags.
- **`virtio-scsi-pci`, nunca "VirtIO SCSI Single":** con Single, el bootstrap se cuelga.
- **`cache=writethrough`** en el disco.
- **Ballooning desactivado** (`--balloon 0`).
- **`--agent enabled=1`:** solo porque la imagen trae `qemu-guest-agent`. Sin la extensión, habilitarlo solo llena el log.

> La doc oficial recomienda **8 GB+ de RAM para workers**. Con 6 GB funciona para un
> lab, pero Longhorn consume memoria en cada worker (manager + instance-manager).
>
> El disco de arranque es `scsi0`, y la ISO en `ide2` solo se usa mientras el disco está
> vacío. Después de instalar, Talos arranca del disco solo.

Los tres nodos arrancan en **modo mantenimiento** (sin config). Verificar el disco:

```bash
talosctl get disks --insecure --nodes $CONTROL_PLANE_IP
# debe aparecer sda (~43 GB)
```

---

## 4. Generar la configuración

El patch de Longhorn (`talos/longhorn-patch.yaml`, ya está en el repo) fija el
installer con extensiones y monta `/var/lib/longhorn` dentro del kubelet con
propagación `rshared`:

```yaml
machine:
  install:
    image: factory.talos.dev/metal-installer/53513e54bb39202f35694412577a6bc53d484744d35a126e5d42ef34785c0d83:v1.14.0
  kubelet:
    extraMounts:
      - destination: /var/lib/longhorn
        type: bind
        source: /var/lib/longhorn
        options:
          - bind
          - rshared
          - rw
```

> Si cambiás `TALOS_VERSION` o el schematic, actualizá también `install.image` en este archivo.

```bash
talosctl gen config $CLUSTER_NAME https://$CONTROL_PLANE_IP:6443 \
  --output-dir _out \
  --install-disk /dev/sda \
  --config-patch @talos/longhorn-patch.yaml
```

Genera `_out/controlplane.yaml`, `_out/worker.yaml` y `_out/talosconfig`.

> `_out/` está en el `.gitignore`: tiene las claves de la CA del cluster, de etcd y
> el secreto de cifrado. **Nunca commitearlo.** Guardá una copia fuera de la máquina.

Fijar endpoint y nodo por defecto en el talosconfig, así no hace falta `-n` en cada comando:

```bash
talosctl config endpoint $CONTROL_PLANE_IP
talosctl config node $CONTROL_PLANE_IP
```

---

## 5. Control plane

Aplicar la config con el hostname:

```bash
talosctl apply-config --insecure --nodes $CONTROL_PLANE_IP --file _out/controlplane.yaml --config-patch '{"apiVersion":"v1alpha1","kind":"HostnameConfig","auto":"off","hostname":"controlplane1"}'
```

El nodo instala Talos en `/dev/sda` y reinicia. Esperá a que vuelva a responder
(1–2 minutos) y hacé el bootstrap de etcd **una sola vez**:

```bash
talosctl bootstrap --nodes $CONTROL_PLANE_IP
```

Kubeconfig (se fusiona en `~/.kube/config`):

```bash
talosctl kubeconfig --nodes $CONTROL_PLANE_IP
kubectl get nodes
```

---

## 6. Workers

Cada comando en **una sola línea**: el JSON del patch no se puede partir.

```bash
talosctl apply-config --insecure --nodes $WORKER1_IP --file _out/worker.yaml --config-patch '{"apiVersion":"v1alpha1","kind":"HostnameConfig","auto":"off","hostname":"worker1"}'

talosctl apply-config --insecure --nodes $WORKER2_IP --file _out/worker.yaml --config-patch '{"apiVersion":"v1alpha1","kind":"HostnameConfig","auto":"off","hostname":"worker2"}'
```

Esperar a que se sumen:

```bash
kubectl get nodes -o wide -w
```

---

## 7. Verificar el cluster

```bash
talosctl health

# Las extensiones tienen que estar en los 3 nodos
talosctl -n $CONTROL_PLANE_IP,$WORKER1_IP,$WORKER2_IP get extensions
```

Salida esperada, por nodo:

```
qemu-guest-agent
iscsi-tools
util-linux-tools
schematic          53513e54bb39...
```

`schematic` no es una extensión real: es la huella de la imagen, y su versión es el ID del schematic.

Si las dos réplicas de CoreDNS quedaron en el mismo nodo (pasa porque arrancan antes
de que existan los workers), repartirlas:

```bash
kubectl -n kube-system rollout restart deployment coredns
```

---

## 8. Longhorn

### 8.1 Namespace privilegiado

Longhorn necesita Pod Security `privileged`. Talos aplica `baseline`/`restricted`
por defecto, así que el namespace va con los tres labels (el `warn` y el `audit`
solo silencian advertencias):

```bash
kubectl create namespace longhorn-system
kubectl label namespace longhorn-system \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/warn=privileged \
  pod-security.kubernetes.io/audit=privileged
```

### 8.2 Values

`practice/longhorn-values.yaml` (ya está en el repo):

```yaml
# Solo 2 workers: con 3 réplicas los volúmenes quedarían siempre "degraded"
defaultSettings:
  defaultReplicaCount: 2
persistence:
  defaultClass: true
  defaultClassReplicaCount: 2
preUpgradeChecker:
  jobEnabled: false
```

### 8.3 Instalar

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update longhorn

helm install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --version 1.12.1 \
  -f practice/longhorn-values.yaml \
  --wait --timeout 10m
```

> **No le agregues `| head`, `| grep` ni nada que corte la salida a `helm install`.** Si el
> pipe se cierra antes, Helm muere a mitad de la instalación y el release queda en
> `pending-install`, sin los Deployments del driver CSI ni la UI. Ver *Problemas
> conocidos*.

### 8.4 Verificar

```bash
helm -n longhorn-system list                 # STATUS: deployed
kubectl -n longhorn-system get pods          # todos Running (~23 pods)
kubectl get storageclass                     # longhorn (default)
kubectl get csidriver                        # driver.longhorn.io
kubectl -n longhorn-system get nodes.longhorn.io   # worker1 y worker2, READY True
```

Longhorn corre solo en los workers: el control plane tiene el taint `NoSchedule`.

### 8.5 UI

No está expuesta. Para entrar:

```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
# http://localhost:8080
```

---

## 9. Prueba de punta a punta

`practice/stateful.yaml` crea un StatefulSet con un PVC de 5 Gi en `longhorn`:

```bash
kubectl apply -f practice/stateful.yaml
kubectl rollout status sts/my-csi-app-set --timeout=5m

kubectl get pvc                                   # Bound
kubectl exec my-csi-app-set-0 -- sh -c 'echo hola-longhorn > /data/test && cat /data/test && df -h /data'

kubectl -n longhorn-system get volumes.longhorn.io \
  -o custom-columns=NAME:.metadata.name,STATE:.status.state,ROBUSTNESS:.status.robustness,REPLICAS:.spec.numberOfReplicas
# attached   healthy   2
```

Limpiar (el PVC de un StatefulSet no se borra solo):

```bash
kubectl delete -f practice/stateful.yaml
kubectl delete pvc csi-pvc-my-csi-app-set-0
```

---

## 10. Después de instalar

Snapshot de etcd. Con un solo control plane es la única red de contención:

```bash
talosctl -n $CONTROL_PLANE_IP etcd snapshot _out/etcd-$(date +%F).snapshot
```

Restaurar (sobre un control plane recién instalado, en lugar de `bootstrap`):

```bash
talosctl -n $CONTROL_PLANE_IP bootstrap --recover-from _out/etcd-AAAA-MM-DD.snapshot
```

---

## Anexo A — Cambiar extensiones o versión en un cluster andando

No hace falta reinstalar: `talosctl upgrade` reescribe el sistema y conserva `STATE`
(la config) y `EPHEMERAL` (`/var`, que incluye `/var/lib/longhorn`).

```bash
# 1. Snapshot de etcd (paso 10)
# 2. Nuevo schematic → nuevo ID → actualizar install.image en talos/longhorn-patch.yaml
# 3. Aplicar el patch (no reinicia)
for n in $CONTROL_PLANE_IP $WORKER1_IP $WORKER2_IP; do
  talosctl -n $n patch machineconfig --patch @talos/longhorn-patch.yaml
done

# 4. Upgrade de a un nodo, workers primero
IMG=factory.talos.dev/metal-installer/$SCHEMATIC:$TALOS_VERSION
for n in $WORKER1_IP $WORKER2_IP $CONTROL_PLANE_IP; do
  talosctl -n $n upgrade --image $IMG --wait --timeout 10m
done
```

Algunos detalles:

- **Los nodos se actualizan de a uno.** `upgrade` hace cordon + drain, reinicia y después uncordon.
- **Con Longhorn, esperá entre nodos.** Antes de pasar al siguiente, los volúmenes tienen que volver a `healthy`. Si reiniciás el otro worker mientras un volumen se reconstruye, se pierde la réplica sana.
- **`--preserve` ya no hace falta.** Desde Talos v1.8 se aplica solo. En guías viejas figura como obligatorio: sin ese flag se borraba `/var/lib/longhorn`.
- **Ctrl-C no cancela el upgrade.** Solo corta el seguimiento del cliente. El nodo termina igual, pero puede quedar cordoneado: arreglalo con `kubectl uncordon <nodo>`.
- **Con un solo control plane, la API se cae mientras ese nodo reinicia.**

---

## Anexo B — Variante recomendada por Longhorn: disco dedicado (`UserVolumeConfig`)

Esta guía guarda los datos de Longhorn en `/var/lib/longhorn`, que vive en la partición
`EPHEMERAL` del disco del sistema: es lo más simple y así está el cluster hoy.
Para Talos v1.10+, la doc de Longhorn recomienda un volumen dedicado montado en
`/var/mnt/longhorn`. Así los datos de Longhorn no compiten con imágenes y logs por el
mismo espacio.

1. Agregar un segundo disco a cada worker en Proxmox, por ejemplo de 100 GB:
   ```bash
   qm set 211 --scsi1 local-lvm:100,cache=writethrough,discard=on,ssd=1
   qm set 212 --scsi1 local-lvm:100,cache=writethrough,discard=on,ssd=1
   ```
2. Patch para los workers (`talos/longhorn-volume.yaml`):
   ```yaml
   apiVersion: v1alpha1
   kind: UserVolumeConfig
   name: longhorn          # se monta en /var/mnt/longhorn
   provisioning:
     diskSelector:
       match: '!system_disk'
     minSize: 50GB
     grow: true
   ```
3. En `talos/longhorn-patch.yaml`, cambiar `/var/lib/longhorn` por `/var/mnt/longhorn`
   (en `source` y `destination`).
4. Generar la config sumando el patch nuevo:
   `--config-patch-worker @talos/longhorn-volume.yaml`.
5. En `practice/longhorn-values.yaml` agregar:
   ```yaml
   defaultSettings:
     defaultDataPath: /var/mnt/longhorn
   ```

---

## Problemas conocidos

| Síntoma | Causa | Solución |
|---|---|---|
| El bootstrap se cuelga | Controladora `VirtIO SCSI Single` | Usar `--scsihw virtio-scsi-pci` |
| PVC en `Pending` | No hay StorageClass o el nombre no existe (p. ej. `do-block-storage`, que es de DigitalOcean) | `storageClassName: longhorn` o dejarlo vacío (usa el default) |
| Helm en `pending-install`, faltan `longhorn-driver-deployer`, `csi-*` y `longhorn-ui` | `helm install` interrumpido | Si todavía **no hay volúmenes**: `kubectl -n longhorn-system delete secret sh.helm.release.v1.longhorn.v1` y repetir el `helm upgrade --install ... --wait` |
| Pods de Longhorn rechazados por PodSecurity | Falta el label `enforce=privileged` | Paso 8.1 |
| Volúmenes siempre `degraded` | 3 réplicas con 2 workers | `defaultReplicaCount: 2` |
| `kube-controller-manager` con restarts después de un reboot | Arranca antes que el apiserver (`EOF` en `127.0.0.1:7445`) | Normal, se recupera solo |
| `kubectl top` → `Metrics API not available` | No hay metrics-server | Opcional, no es parte de este setup |
