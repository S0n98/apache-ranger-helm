# Apache Ranger Helm Chart

A Helm chart that deploys [Apache Ranger](https://ranger.apache.org/) Admin (v2.8.0)
with a bundled [Bitnami PostgreSQL](https://github.com/bitnami/charts/tree/main/bitnami/postgresql)
database as its policy store.

- **Chart version:** `0.2.0`
- **App version (Ranger):** `2.8.0`
- **Database:** PostgreSQL 14 (bundled subchart, `postgresql` v13.2.27)

---

## Contents

```
.
├── apache-ranger/                 # Chart source
│   ├── Chart.yaml                 # version 0.2.0 / appVersion 2.8.0
│   ├── values.yaml                # default values (ingress disabled by default)
│   ├── charts/postgresql/         # bundled Bitnami PostgreSQL subchart
│   └── templates/
│       ├── deployment.yaml        # Ranger Admin Deployment
│       ├── configmap.yaml         # Ranger install.properties
│       ├── service.yaml           # ClusterIP service on :6080
│       ├── ingress.yaml           # optional Ingress
│       └── NOTES.txt              # post-install access instructions
├── apache-ranger-0.2.0.tgz        # packaged chart (this version)
├── apache-ranger-0.1.1.tgz        # previous packaged chart
└── ranger-values.yaml             # example/override values (ingress enabled, ranger.local)
```

---

## Quick start

```bash
# from this directory
helm upgrade --install ranger ./apache-ranger \
  --namespace ranger --create-namespace \
  -f ranger-values.yaml
```

> First-time startup takes **several minutes**: PostgreSQL initializes, then Ranger
> imports its DB schema (~78 tables) before the admin UI comes up. Follow progress with
> `kubectl logs -n ranger -l app=apache-ranger -f`.

### Default credentials

| Setting  | Value                     |
| -------- | ------------------------- |
| Username | `admin`                   |
| Password | `ranger.adminPassword` (default `StrongPass58`) |

---

## Accessing the UI

### Via Ingress (NGINX)

`ranger-values.yaml` enables an Ingress on host **`ranger.local`** using the `nginx`
IngressClass:

```yaml
ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
  hosts:
    - host: ranger.local
      paths:
        - path: /
          pathType: Prefix
  tls: []
```

Point `ranger.local` at your ingress controller's address (e.g. the node IP for an
RKE2 / hostNetwork ingress-nginx), then browse to `http://ranger.local/`:

```bash
# /etc/hosts  (replace with your ingress controller address)
192.168.2.100  ranger.local
```

Verify routing without DNS:

```bash
curl -H "Host: ranger.local" http://<ingress-address>/login.jsp   # -> 200
```

### Via port-forward (no Ingress)

```bash
kubectl port-forward -n ranger svc/ranger-apache-ranger 6080:6080
# then open http://localhost:6080/
```

---

## Configuration

| Key | Default | Description |
| --- | --- | --- |
| `replicaCount` | `1` | Ranger Admin replicas |
| `image.repository` / `image.tag` | `apache/ranger` / `2.8.0` | Ranger Admin image |
| `ranger.adminPassword` | `StrongPass58` | Password for `admin`, `keyadmin`, tagsync, usersync |
| `service.type` / `service.port` | `ClusterIP` / `6080` | Ranger Admin service |
| `ingress.enabled` | `false` (chart) / `true` (`ranger-values.yaml`) | Create an Ingress |
| `ingress.className` | `nginx` | IngressClass name |
| `ingress.hosts` | `ranger.local` | Ingress hosts/paths |
| `resources` | 250m–500m CPU / 512Mi–1Gi | Ranger Admin resources |
| `postgresql.enabled` | `true` | Deploy the bundled PostgreSQL |
| `postgresql.image.repository` | `bitnamilegacy/postgresql` | DB image (see note below) |
| `postgresql.auth.postgresPassword` | `postgrespass` | `postgres` superuser password (used by Ranger as DB root) |
| `postgresql.auth.username` / `password` | `ranger` / `rangerpass` | Ranger application DB user |
| `postgresql.auth.database` | `rangerdb` | Ranger database name |
| `postgresql.primary.startupProbe` | enabled, ~10 min budget | Protects slow first-time DB init (see fixes) |
| `postgresql.primary.persistence.size` | `8Gi` | DB volume size |

### Database wiring

Ranger connects to the bundled PostgreSQL using these values (rendered into
`install.properties`):

- `db_host = <release>-postgresql`
- `db_name = postgresql.auth.database` (`rangerdb`)
- `db_root_user = postgres`, `db_root_password = postgresql.auth.postgresPassword`
- `db_user = postgresql.auth.username`, `db_password = postgresql.auth.password`

---

## How this chart works (notable design points)

This chart deviates from the stock `apache/ranger` image entrypoint to get a reliable
deployment. The reasons are documented inline in `templates/deployment.yaml`:

1. **It does not run the image's `/home/ranger/scripts/ranger.sh`.** That entrypoint
   overwrites `install.properties` with image defaults (`db_host=ranger-db`,
   `db_name=ranger`) and depends on env vars this chart doesn't set, which breaks the DB
   connection. Instead the container waits for PostgreSQL, then runs `setup.sh` directly
   against the chart-provided `install.properties` (correct host/db/user), starts the
   admin service, and keeps the container alive on the admin PID.
2. **`setup.sh` is mounted from the ConfigMap with the `chown -R ... $INSTALL_DIR` lines
   neutralized** (via an initContainer `sed`), because the container runs as root.
3. **`configmap.yaml` includes every property the image's default template defines**
   (including the Kerberos/SPNEGO and audit-JAAS keys, left empty). `setup.sh` injects a
   "property not found" *error string* as the value for any missing key, which otherwise
   corrupts numeric config such as `ranger.admin.kerberos.token.valid.seconds`.
4. **A `checksum/config` pod annotation** rolls the Deployment automatically whenever the
   ConfigMap changes.

---

## Troubleshooting / fixes baked into 0.2.0

This version fixes three stacked failures seen on a fresh install (notably on slow,
NFS-backed storage):

### 1. `password authentication failed for user "postgres"` / *User postgres has no password assigned*
**Cause:** On slow storage, PostgreSQL `initdb` takes longer than the Bitnami default
`livenessProbe` tolerates (~90–120s). The container was SIGKILLed **mid-init**, before
the postgres/ranger passwords were assigned. On restart it found a half-written volume
and skipped credential setup permanently — so Ranger looped on "PostgreSQL not ready".

**Fix:** `postgresql.primary.startupProbe` is enabled with a generous budget
(`failureThreshold: 40`, `periodSeconds: 15` ≈ 10 min) and `livenessProbe` is relaxed, so
liveness/readiness are gated until init completes.

> If you already have a corrupted PVC from a previous failed install, delete it so the DB
> reinitializes cleanly:
> ```bash
> kubectl delete pvc data-ranger-postgresql-0 -n ranger
> kubectl delete pod ranger-postgresql-0 -n ranger
> ```

### 2. Ranger connecting to `ranger-db` with an empty user
**Cause:** the image entrypoint clobbered the mounted `install.properties`.
**Fix:** run `setup.sh` directly with the chart's config (see design point #1).

### 3. `NumberFormatException` on Ranger Admin startup
**Cause:** properties missing from `install.properties` caused `setup.sh` to write a
"not found" error string into `ranger.admin.kerberos.token.valid.seconds`.
**Fix:** all image-default keys are now present in `configmap.yaml` (design point #3).

### Image note
The PostgreSQL image is set to `bitnamilegacy/postgresql` because Bitnami moved its free
Debian images to the `bitnamilegacy` namespace; the original `bitnami/postgresql:16.x`
reference no longer pulls.

---

## Uninstall

```bash
helm uninstall ranger -n ranger
# PVCs are retained by default — delete to remove DB data:
kubectl delete pvc -n ranger -l app.kubernetes.io/instance=ranger
```
