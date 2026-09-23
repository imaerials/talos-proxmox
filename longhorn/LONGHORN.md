# Longhorn en Talos

Almacenamiento en bloque replicado para los `PersistentVolumeClaim` del cluster, sobre el
segundo disco de cada worker.

Referencia: <https://longhorn.io/docs/1.12.1/advanced-resources/os-distro-specific/talos-linux-support/>

| | |
|---|---|
| Chart | `longhorn/longhorn` 1.12.1 |
| Disco de datos | `/dev/sdb` de cada worker, montado en `/var/mnt/longhorn` (`talos/longhorn-user-disk.yaml`) |
| Réplicas | 2 (una por worker) |
| StorageClass | `longhorn` (default) |
| UI | No expuesta (port-forward) |

## Archivos

| Archivo | Contenido |
|---|---|
| `namespace.yaml` | Namespace `longhorn-system` con pod security `privileged` |
| `values.yaml` | Values de Helm: ruta de datos y cantidad de réplicas |

Todos los comandos se corren desde la raíz del repo.

## 0. Requisitos

- Imagen de Talos con las extensiones `iscsi-tools` y `util-linux-tools` (la del paso 2.2 de
  `Setup-Final.md` ya las trae).
- Disco de datos montado en los workers (paso 5.1 de `Setup-Final.md`):

  ```bash
  talosctl -n $WORKER1_IP,$WORKER2_IP get volumestatus u-longhorn
  ```

## 1. Namespace

```bash
kubectl apply -f longhorn/namespace.yaml
```

## 2. Instalar Longhorn

```bash
helm repo add longhorn https://charts.longhorn.io && helm repo update
helm install longhorn longhorn/longhorn -n longhorn-system --version 1.12.1 \
  -f longhorn/values.yaml
```

Para cambiar la config después, editar `longhorn/values.yaml` y:

```bash
helm upgrade longhorn longhorn/longhorn -n longhorn-system --version 1.12.1 \
  -f longhorn/values.yaml
```

`defaultSettings` solo se aplica en la instalación. Si Longhorn ya está corriendo, los cambios
ahí se hacen desde la UI o con `kubectl -n longhorn-system edit settings.longhorn.io <nombre>`.

## 3. Verificar

```bash
kubectl -n longhorn-system rollout status deploy/longhorn-driver-deployer
kubectl get nodes.longhorn.io -n longhorn-system   # 2 workers, SCHEDULABLE=true
kubectl get storageclass                           # longhorn (default)
```

## 4. UI

```bash
kubectl -n longhorn-system port-forward svc/longhorn-frontend 8080:80
# http://localhost:8080
```

## Problemas comunes

| Síntoma | Causa probable |
|---|---|
| Pods de `longhorn-manager` en `CrashLoopBackOff` | Falta la extensión `iscsi-tools` en la imagen de Talos |
| Pods rechazados por pod security | Falta el label `privileged` en el namespace (paso 1) |
| Nodo con `SCHEDULABLE=false` o sin disco | `/var/mnt/longhorn` no está montado: revisar `volumestatus u-longhorn` |
| Volumen en `Degraded` | Un worker no está disponible: con 2 réplicas y 2 workers no hay dónde reconstruirla hasta que vuelva |
