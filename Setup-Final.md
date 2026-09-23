## 0. Variables

Todo el documento usa estas variables. Se corre desde la raíz del repo.

```bash
cd ~/talos-proxmox

export CLUSTER_NAME=talos-proxmox-cluster
export CONTROL_PLANE_IP=192.168.68.83
export WORKER1_IP=192.168.68.82
export WORKER2_IP=192.168.68.84
export TALOSCONFIG="$PWD/_out/talosconfig"
```

## 1. Crear VMs

En el host Proxmox. Los workers llevan un segundo disco (`scsi1`) dedicado a Longhorn;
el control plane solo lleva el disco del sistema.

```bash
STORAGE=local-lvm
ISO=local:iso/talos-v1.14.1-ext-metal-amd64.iso
BRIDGE=vmbr0

# data_disk (GB) es opcional: si se pasa, agrega scsi1 para Longhorn
create_talos_vm() {
  local vmid=$1 name=$2 cores=$3 mem=$4 disk=$5 data_disk=${6:-}
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
    --serial0 socket

  if [ -n "$data_disk" ]; then
    qm set "$vmid" --scsi1 ${STORAGE}:${data_disk},discard=on,ssd=1,backup=0
  fi
}

#               vmid name         cores mem  disco longhorn
create_talos_vm 210  talos-cp-01  2     4096 30
create_talos_vm 211  talos-w-01   4     6144 30    100
create_talos_vm 212  talos-w-02   4     6144 30    100
```

- `scsi0` sigue siendo el disco del sistema (Talos se instala en `/dev/sda`); el de Longhorn aparece como `/dev/sdb`.
- `backup=0` excluye el disco de Longhorn de los backups de Proxmox: los datos ya están replicados por Longhorn.
- Mantener `virtio-scsi-pci` (VirtIO SCSI); `virtio-scsi-single` cuelga el bootstrap de Talos.
- Verificar (con los nodos en maintenance mode): `talosctl get disks --insecure --nodes $WORKER1_IP` debe listar `sda` (~32 GB) y `sdb` (~107 GB).

## 2. Generar la configuración

Genera la config de todos los nodos (control plane y workers).

### 2.1 Patch de Longhorn para los workers

Monta el segundo disco (el que no es de sistema) en `/var/mnt/longhorn`:

```bash
mkdir -p talos
cat > talos/longhorn-user-disk.yaml <<'YAML'
apiVersion: v1alpha1
kind: UserVolumeConfig
name: longhorn
provisioning:
  diskSelector:
    match: "!system_disk"
  minSize: 50GB
  grow: true
YAML
```

### 2.2 Generar

```bash
talosctl gen config $CLUSTER_NAME https://$CONTROL_PLANE_IP:6443 \
  --output-dir _out \
  --install-image factory.talos.dev/metal-installer/88d1f7a5c4f1d3aba7df787c448c1d3d008ed29cfb34af53fa0df4336a56040b:v1.14.1 \
  --config-patch-worker @talos/longhorn-user-disk.yaml
```

## 3. Aplicar la configuración

### 3.1 Control plane

```bash
talosctl apply-config --insecure --nodes $CONTROL_PLANE_IP --file _out/controlplane.yaml
```

### 3.2 Workers

```bash
for ip in $WORKER1_IP $WORKER2_IP; do
  talosctl apply-config --insecure --nodes $ip --file _out/worker.yaml
done
```

## 4. Bootstrap the cluster

### 4.1 Configurar talosctl

```bash
export TALOSCONFIG="_out/talosconfig"
talosctl config endpoint $CONTROL_PLANE_IP
talosctl config node $CONTROL_PLANE_IP
```

### 4.2 Bootstrap etcd

```bash
talosctl bootstrap
```

### 4.3 Retrieve the kubeconfig

```bash
talosctl kubeconfig -e $CONTROL_PLANE_IP
```

### 4.4 Verify that your cluster is ready

```bash
kubectl get nodes
```

## 5. Longhorn

### 5.1 Verificar el disco

El volumen `u-longhorn` debe quedar "ready" y montado en `/var/mnt/longhorn`:

```bash
talosctl -n $WORKER1_IP,$WORKER2_IP get volumestatus u-longhorn
talosctl -n $WORKER1_IP,$WORKER2_IP get mountstatus | grep longhorn
```

Si la config se generó sin el patch del paso 2.1, aplicarlo ahora:

```bash
talosctl -n $WORKER1_IP,$WORKER2_IP patch machineconfig --patch @talos/longhorn-user-disk.yaml
```

### 5.2 Instalar

Namespace, Helm y verificación: ver `longhorn/LONGHORN.md`.
