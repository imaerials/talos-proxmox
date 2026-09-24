# Ingress (Traefik) + cert-manager en Talos

Acceso por nombre (`app.lab`) y con HTTPS a los servicios del cluster, usando la IP fija
que da MetalLB (ver `metalLb/METALLB.md`).

| | |
|---|---|
| Ingress controller | Traefik, chart `traefik/traefik` 41.6.0 (Traefik v3.7.13) |
| Certificados | cert-manager, chart `jetstack/cert-manager` v1.21.2 |
| IP del ingress | `192.168.68.200` (primera del pool de MetalLB, fijada con anotación) |
| Emisor de certificados | CA propia del lab (`lab-ca`); Let's Encrypt opcional (ver final) |
| Nombres | `*.192.168.68.200.nip.io` (no requiere DNS propio) |

¿Por qué Traefik y no ingress-nginx? El proyecto Kubernetes retiró ingress-nginx: dejó de
recibir mantenimiento y parches de seguridad en marzo de 2026.

Todos los comandos se corren desde la raíz del repo.

## Flujo

```
cliente ──https──▶ 192.168.68.200 (MetalLB L2) ──▶ Traefik ──▶ Service ──▶ Pod
                                                     ▲
                                  Secret TLS ◀── cert-manager (ClusterIssuer lab-ca)
```

## 1. cert-manager

### 1.1 Instalar

```bash
helm repo add jetstack https://charts.jetstack.io && helm repo update
helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace --version v1.21.2 \
  -f ingress/cert-manager-values.yaml
```

`ingress/cert-manager-values.yaml` instala los CRDs y corre controller, webhook y cainjector
con 2 réplicas, una por worker, cada uno con su PodDisruptionBudget (ver [HA](#ha)).

```bash
kubectl -n cert-manager rollout status deploy/cert-manager-webhook
kubectl -n cert-manager get pods
```

### 1.2 CA propia del lab

Archivo `ingress/cluster-issuers.yaml`, con tres recursos: un emisor autofirmado (`selfsigned`)
que solo sirve para crear la CA, el certificado de la CA (`lab-ca`, 10 años) y el
`ClusterIssuer` `lab-ca` que firma los certificados de las apps.

```bash
kubectl apply -f ingress/cluster-issuers.yaml
```

Si falla con un error del webhook, cert-manager todavía está arrancando: esperar y repetir.

```bash
kubectl get clusterissuers           # selfsigned y lab-ca en READY=True
kubectl -n cert-manager get certificate lab-ca
```

### 1.3 Confiar en la CA (en cada cliente)

Para que el navegador no marque los certificados como inválidos:

```bash
kubectl -n cert-manager get secret lab-ca -o jsonpath='{.data.ca\.crt}' | base64 -d > lab-ca.crt
```

- **Linux/WSL:** `sudo cp lab-ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates`
- **Windows:** doble clic en `lab-ca.crt` → Instalar certificado → "Entidades de certificación raíz de confianza".
- **Firefox:** usa su propio almacén: Ajustes → Certificados → Importar.

`lab-ca.crt` es público, pero **no** subir el Secret `lab-ca` completo: tiene la clave privada.

## 2. Traefik

### 2.1 Values

Archivo `ingress/traefik-values.yaml`:

- IP fija `192.168.68.200` de MetalLB (anotación `metallb.io/loadBalancerIPs`).
- Traefik como `IngressClass` por defecto.
- Redirección permanente de http a https.
- Dashboard en `traefik.192.168.68.200.nip.io` (ver [Dashboard](#dashboard)).
- 2 réplicas, una por worker, con PodDisruptionBudget (ver [HA](#ha)).

### 2.2 Instalar

```bash
helm repo add traefik https://traefik.github.io/charts && helm repo update
helm install traefik traefik/traefik \
  -n traefik --create-namespace --version 41.6.0 \
  -f ingress/traefik-values.yaml
```

```bash
kubectl -n traefik rollout status deploy/traefik
kubectl -n traefik get svc traefik   # EXTERNAL-IP = 192.168.68.200
kubectl get ingressclass             # traefik (default)
```

### Dashboard

<https://traefik.192.168.68.200.nip.io/dashboard/> (la `/` final es necesaria). Lo publica
un `IngressRoute` que crea el chart (`ingressRoute.dashboard`) en el entrypoint `websecure`,
con el certificado por defecto de Traefik (autofirmado: el navegador pide aceptar la excepción).

```bash
kubectl -n traefik get ingressroute traefik-dashboard
curl -k https://traefik.192.168.68.200.nip.io/api/version
```

No tiene autenticación: cualquiera en la LAN ve la configuración de ruteo (es de solo
lectura). Para no exponerlo, poner `ingressRoute.dashboard.enabled: false` y entrar con
port-forward en <http://localhost:8080/dashboard/>:

```bash
kubectl -n traefik port-forward deploy/traefik 8080:8080
```

## 3. Probar

Archivo `ingress/test-whoami.yaml`: app `whoami` con un `Ingress` que pide su certificado a
`lab-ca` para `whoami.192.168.68.200.nip.io`.

```bash
kubectl apply -f ingress/test-whoami.yaml
```

```bash
kubectl get certificate whoami-tls          # READY=True
curl -I  http://whoami.192.168.68.200.nip.io              # 301 → https
curl --cacert lab-ca.crt https://whoami.192.168.68.200.nip.io
```

Limpiar:

```bash
kubectl delete -f ingress/test-whoami.yaml
kubectl delete secret whoami-tls   # cert-manager no lo borra con el Ingress
```

## 4. Uso en una app

Solo hace falta un `Ingress` con la anotación del issuer y un bloque `tls`:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: lab-ca
spec:
  tls:
    - hosts: [miapp.192.168.68.200.nip.io]
      secretName: miapp-tls
  rules:
    - host: miapp.192.168.68.200.nip.io
      ...
```

cert-manager crea el `Certificate`, lo guarda en `miapp-tls` y lo renueva solo.

### Nombres sin nip.io

`nip.io` resuelve `<algo>.192.168.68.200.nip.io` a `192.168.68.200` usando DNS público.
Alternativas si no se quiere depender de eso:

- `/etc/hosts` (o `C:\Windows\System32\drivers\etc\hosts`): `192.168.68.200  whoami.lab`
- Un registro comodín `*.lab → 192.168.68.200` en el DNS del router o en Pi-hole/AdGuard.

## Opcional: Let's Encrypt

Sirve solo si tenés un dominio propio con DNS en un proveedor con API (Cloudflare, Route 53, …).
Como el cluster no está expuesto a internet, el desafío tiene que ser **DNS-01**
(HTTP-01 no funciona). Se agrega como otro `ClusterIssuer` y se elige por app con la anotación
`cert-manager.io/cluster-issuer`. Doc: <https://cert-manager.io/docs/configuration/acme/dns01/>

## Problemas comunes

| Síntoma | Causa probable |
|---|---|
| `EXTERNAL-IP` de Traefik en `<pending>` | MetalLB no instalado, o `192.168.68.200` ya asignada a otro Service |
| `apply` de los issuers falla con error de webhook | cert-manager todavía arrancando: esperar y repetir |
| `Certificate` en `READY=False` | `kubectl describe certificate <nombre>` y `kubectl get certificaterequests,orders -A` |
| `nip.io` no resuelve | El router bloquea respuestas DNS con IPs privadas (DNS rebinding protection): usar `/etc/hosts` o DNS local |
| El navegador marca el certificado inválido | Falta importar `lab-ca.crt` en ese cliente (paso 1.3) |
| 404 de Traefik | El `host` del Ingress no coincide con la URL, o falta `ingressClassName` y Traefik no es la clase por defecto |

## HA

Traefik y cert-manager corren 2 réplicas repartidas entre los dos workers con
`topologySpreadConstraints`:

- `whenUnsatisfiable: DoNotSchedule`: con `ScheduleAnyway` el scoring por recursos
  gana y el scheduler puede apilar las dos réplicas en el nodo más vacío.
- `nodeTaintsPolicy: Honor`: el control plane (taint `NoSchedule`) y un worker caído
  (taint `unreachable`) no cuentan como destino; si cae un worker, la réplica se
  reprograma en el otro en vez de quedar Pending.
- `matchLabelKeys: [pod-template-hash]`: cada revisión se reparte por separado, así
  el pod extra del rolling update (`maxSurge`) no queda Pending.

Kubernetes no rebalancea pods que ya están corriendo: si un worker arranca tarde
(p. ej. después de reiniciar el host), lo que se programó mientras tanto queda en el
otro. Para Deployments sin estas reglas se corrige con `kubectl rollout restart`.

Verificar el reparto:

```bash
kubectl get pods -n traefik -o wide
kubectl get pods -n cert-manager -o wide
kubectl get pdb -A
```
