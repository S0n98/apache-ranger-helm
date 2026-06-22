# Apache Ranger Helm Chart

A Helm chart that deploys [Apache Ranger](https://ranger.apache.org/) Admin (v2.8.0)
with a bundled [Bitnami PostgreSQL](https://github.com/bitnami/charts/tree/main/bitnami/postgresql)
database as its policy store.

- **Chart version:** `0.6.0`
- **App version (Ranger):** `2.8.0`
- **Database:** PostgreSQL 14 (bundled subchart, `postgresql` v13.2.27)

---

## Contents

```
.
├── apache-ranger/                 # Chart source
│   ├── Chart.yaml                 # version 0.6.0 / appVersion 2.8.0
│   ├── values.yaml                # default values (ingress disabled by default)
│   ├── charts/postgresql/         # bundled Bitnami PostgreSQL subchart
│   └── templates/
│       ├── deployment.yaml        # Ranger Admin Deployment
│       ├── configmap.yaml         # Ranger install.properties
│       ├── service.yaml           # ClusterIP service on :6080
│       ├── ingress.yaml           # optional Ingress
│       └── NOTES.txt              # post-install access instructions
├── apache-ranger-0.5.0.tgz        # previous packaged chart (customCaCerts keytool fix)
├── apache-ranger-0.4.0.tgz        # previous packaged chart (customCaCerts)
├── apache-ranger-0.3.0.tgz        # previous packaged chart (hostAliases)
├── apache-ranger-0.2.0.tgz        # previous packaged chart
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

## Values reference

All configurable values for this chart. Override them with `-f ranger-values.yaml` or
`--set key=value`.

### General

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `replicaCount` | int | `1` | Number of Ranger Admin pod replicas |

### Image

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `image.repository` | string | `apache/ranger` | Ranger Admin container image |
| `image.tag` | string | `"2.8.0"` | Image tag |
| `image.pullPolicy` | string | `IfNotPresent` | Kubernetes image pull policy (`Always`, `IfNotPresent`, `Never`) |

### Ranger

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `ranger.adminPassword` | string | `"StrongPass58"` | Shared password for the `admin`, `keyadmin`, `rangerTagsync`, and `rangerUsersync` users |

### Host aliases

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `hostAliases` | list | `[]` | Pod-level `/etc/hosts` entries. Each item has `ip` (string) and `hostnames` (list of strings) |

Add custom entries to the pod's `/etc/hosts` — useful when Ranger needs to reach
services by hostname that are not resolvable via cluster DNS:

```yaml
hostAliases:
  - ip: "10.0.0.50"
    hostnames:
      - "trino.internal"
  - ip: "10.0.0.51"
    hostnames:
      - "hive-metastore.internal"
```

### Custom CA certificates (TLS trust)

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `customCaCerts.enabled` | bool | `false` | Mount a Secret of CA certificates and import them into the JVM truststore at startup |
| `customCaCerts.secretName` | string | `""` | Name of a Kubernetes Secret whose data keys are PEM (`.crt` / `.pem`) certificate files |

If Ranger connects to services over TLS with certificates signed by a private CA
(e.g. Trino, Hive, Solr), the JVM will reject the connection with a
`PKIX path building failed` error. To fix this, provide the CA certificate(s) in a
Kubernetes Secret and enable `customCaCerts`:

```bash
# 1. Extract the CA certificate (example: self-signed Trino behind ingress)
openssl s_client -connect trino.local:443 -showcerts </dev/null 2>/dev/null \
  | openssl x509 -outform PEM > trino-ca.crt

# 2. Create a Kubernetes secret with the CA cert(s)
kubectl create secret generic trino-ca-cert \
  --from-file=trino-ca.crt=trino-ca.crt \
  -n ranger
```

```yaml
# 3. Enable in values
customCaCerts:
  enabled: true
  secretName: "trino-ca-cert"
```

At container startup, every `.crt` / `.pem` file in the secret is imported into the
JVM's `cacerts` truststore via `keytool` before Ranger starts. You can include
multiple certificate files in a single Secret to trust several CAs.

### Service

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `service.type` | string | `ClusterIP` | Kubernetes Service type (`ClusterIP`, `NodePort`, `LoadBalancer`) |
| `service.port` | int | `6080` | Ranger Admin HTTP port |
| `service.externalIPs` | list | `[]` | List of external IPs to assign to the service |

### Ingress

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `ingress.enabled` | bool | `false` | Create an Ingress resource |
| `ingress.className` | string | `"nginx"` | IngressClass name |
| `ingress.annotations` | object | *(see below)* | Ingress annotations |
| `ingress.hosts` | list | *(see below)* | List of hosts, each with `host` and `paths` (list of `{path, pathType}`) |
| `ingress.tls` | list | `[]` | TLS configuration (list of `{secretName, hosts}`) |

Default annotations:

```yaml
ingress:
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "600"
```

Default hosts:

```yaml
ingress:
  hosts:
    - host: ranger.local
      paths:
        - path: /
          pathType: Prefix
```

### Health probes

Ranger Admin takes several minutes to start (DB schema setup, service initialization).
The chart includes a **startupProbe** that gates liveness and readiness checks until
Ranger is fully serving, preventing premature restarts during the init window.

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `startupProbe.enabled` | bool | `true` | Enable startup probe (gates liveness/readiness) |
| `startupProbe.httpGet.path` | string | `/login.jsp` | HTTP path to probe |
| `startupProbe.httpGet.port` | int | `6080` | Port to probe |
| `startupProbe.initialDelaySeconds` | int | `60` | Seconds before first probe |
| `startupProbe.periodSeconds` | int | `10` | Seconds between probes |
| `startupProbe.timeoutSeconds` | int | `5` | Seconds before a probe times out |
| `startupProbe.failureThreshold` | int | `30` | Failures before giving up (~6 min budget) |
| `livenessProbe.enabled` | bool | `true` | Enable liveness probe (restarts unresponsive pods) |
| `livenessProbe.httpGet.path` | string | `/login.jsp` | HTTP path to probe |
| `livenessProbe.httpGet.port` | int | `6080` | Port to probe |
| `livenessProbe.periodSeconds` | int | `30` | Seconds between probes |
| `livenessProbe.timeoutSeconds` | int | `5` | Seconds before a probe times out |
| `livenessProbe.failureThreshold` | int | `3` | Failures before restarting |
| `readinessProbe.enabled` | bool | `true` | Enable readiness probe (removes from Service endpoints) |
| `readinessProbe.httpGet.path` | string | `/login.jsp` | HTTP path to probe |
| `readinessProbe.httpGet.port` | int | `6080` | Port to probe |
| `readinessProbe.periodSeconds` | int | `10` | Seconds between probes |
| `readinessProbe.timeoutSeconds` | int | `5` | Seconds before a probe times out |
| `readinessProbe.failureThreshold` | int | `3` | Failures before marking unready |

### Resources

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `resources.requests.cpu` | string | `250m` | CPU request for Ranger Admin |
| `resources.requests.memory` | string | `512Mi` | Memory request for Ranger Admin |
| `resources.limits.cpu` | string | `500m` | CPU limit for Ranger Admin |
| `resources.limits.memory` | string | `1024Mi` | Memory limit for Ranger Admin |

### PostgreSQL (bundled subchart)

This chart bundles [Bitnami PostgreSQL](https://github.com/bitnami/charts/tree/main/bitnami/postgresql)
v13.2.27 as a dependency. The values below are the subset used by this chart; see the
[upstream values](https://artifacthub.io/packages/helm/bitnami/postgresql/13.2.27) for
the full reference.

| Key | Type | Default | Description |
| --- | ---- | ------- | ----------- |
| `postgresql.enabled` | bool | `true` | Deploy the bundled PostgreSQL instance |
| `postgresql.image.registry` | string | `docker.io` | PostgreSQL image registry |
| `postgresql.image.repository` | string | `bitnamilegacy/postgresql` | PostgreSQL image (see [image note](#image-note)) |
| `postgresql.image.tag` | string | `14.17.0-debian-12-r3` | PostgreSQL image tag |
| `postgresql.auth.postgresPassword` | string | `postgrespass` | `postgres` superuser password (used by Ranger as `db_root_password`) |
| `postgresql.auth.username` | string | `ranger` | Ranger application DB user |
| `postgresql.auth.password` | string | `rangerpass` | Ranger application DB password |
| `postgresql.auth.database` | string | `rangerdb` | Ranger database name |
| `postgresql.primary.startupProbe.enabled` | bool | `true` | Enable startup probe to protect slow first-time init |
| `postgresql.primary.startupProbe.initialDelaySeconds` | int | `30` | Seconds before the first startup probe |
| `postgresql.primary.startupProbe.periodSeconds` | int | `15` | Seconds between startup probes |
| `postgresql.primary.startupProbe.timeoutSeconds` | int | `5` | Seconds before a probe times out |
| `postgresql.primary.startupProbe.failureThreshold` | int | `40` | Failures allowed before giving up (~10 min budget) |
| `postgresql.primary.livenessProbe.enabled` | bool | `true` | Enable liveness probe |
| `postgresql.primary.livenessProbe.initialDelaySeconds` | int | `60` | Seconds before the first liveness probe |
| `postgresql.primary.livenessProbe.periodSeconds` | int | `10` | Seconds between liveness probes |
| `postgresql.primary.livenessProbe.timeoutSeconds` | int | `5` | Seconds before a probe times out |
| `postgresql.primary.livenessProbe.failureThreshold` | int | `6` | Failures allowed before restarting |
| `postgresql.primary.service.type` | string | `ClusterIP` | PostgreSQL Service type |
| `postgresql.primary.service.port` | int | `5432` | PostgreSQL Service port |
| `postgresql.primary.service.externalIPs` | list | `[]` | External IPs for the PostgreSQL service |
| `postgresql.primary.resources.requests.cpu` | string | `250m` | CPU request for PostgreSQL |
| `postgresql.primary.resources.requests.memory` | string | `512Mi` | Memory request for PostgreSQL |
| `postgresql.primary.resources.limits.cpu` | string | `500m` | CPU limit for PostgreSQL |
| `postgresql.primary.resources.limits.memory` | string | `1024Mi` | Memory limit for PostgreSQL |
| `postgresql.primary.persistence.enabled` | bool | `true` | Enable persistent storage for PostgreSQL |
| `postgresql.primary.persistence.storageClass` | string | `""` | Storage class (empty = cluster default) |
| `postgresql.primary.persistence.size` | string | `8Gi` | PVC size |
| `postgresql.primary.persistence.accessModes` | list | `["ReadWriteOnce"]` | PVC access modes |

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

## Troubleshooting / fixes

### Fixes in 0.6.0

#### Ranger Admin health probes (startup / liveness / readiness)
**Problem:** The Ranger Admin container had no health checks. Kubernetes could not
detect whether the service was up, route traffic before it was ready, or restart it
if it became unresponsive.

**Fix:** Added three HTTP probes against `/login.jsp:6080`:
- **startupProbe** — gates liveness/readiness for up to ~6 minutes (`initialDelaySeconds: 60`, `failureThreshold: 30`, `periodSeconds: 10`), giving Ranger time to run `setup.sh` and start the admin service without being killed prematurely.
- **livenessProbe** — restarts the pod if Ranger stops responding (checked every 30s, 3 failures).
- **readinessProbe** — removes the pod from Service endpoints while it is not yet serving (checked every 10s, 3 failures).

All probes are enabled by default and fully configurable via `values.yaml`.

### Fixes in 0.5.0

#### `keytool: command not found` — custom CA certs not imported
**Cause:** The `keytool` binary is not in `$PATH` inside the `apache/ranger` image; it
lives at `/opt/java/openjdk/bin/keytool`.

**Fix:** The cert-import script now discovers `keytool` via `find` (same approach already
used for `cacerts`) and invokes it by absolute path. Errors are no longer suppressed so
import failures are visible in pod logs.

#### `SSLPeerUnverifiedException: Hostname trino.local not verified` — ingress serving wrong certificate
**Cause:** The Trino ingress references a TLS secret (`trino-tls`) that does not exist.
When the secret is missing, NGINX falls back to its built-in **Fake Certificate**
(`CN=Kubernetes Ingress Controller Fake Certificate`, SAN=`ingress.local`). Java's
hostname verifier rejects the connection because `trino.local` is not in the certificate's
Subject Alternative Names.

**Fix (step-by-step):**

1. Generate a self-signed TLS certificate with `trino.local` in the SAN:

   ```bash
   openssl req -x509 -nodes -days 365 \
     -newkey rsa:2048 \
     -keyout trino-tls.key \
     -out trino-tls.crt \
     -subj "/CN=trino.local/O=Trino" \
     -addext "subjectAltName=DNS:trino.local"
   ```

2. Create the TLS secret in the Trino namespace so the ingress picks it up:

   ```bash
   kubectl -n <trino-namespace> create secret tls trino-tls \
     --cert=trino-tls.crt \
     --key=trino-tls.key
   ```

3. Verify the ingress is now serving the correct certificate:

   ```bash
   openssl s_client -connect trino.local:443 -servername trino.local </dev/null 2>/dev/null \
     | openssl x509 -noout -subject -ext subjectAltName
   # Expected: subject=CN = trino.local, O = Trino
   #           X509v3 Subject Alternative Name: DNS:trino.local
   ```

4. Update Ranger's CA trust secret with the new certificate:

   ```bash
   # Extract the cert the ingress is now serving
   openssl s_client -connect trino.local:443 -showcerts </dev/null 2>/dev/null \
     | openssl x509 -outform PEM > trino-ca.crt

   # Recreate the CA secret in the Ranger namespace
   kubectl -n ranger delete secret trino-ca-cert --ignore-not-found
   kubectl -n ranger create secret generic trino-ca-cert \
     --from-file=trino-ca.crt=trino-ca.crt
   ```

5. Restart Ranger to import the updated CA certificate:

   ```bash
   kubectl -n ranger rollout restart deployment ranger-apache-ranger
   kubectl -n ranger rollout status deployment ranger-apache-ranger --timeout=120s
   ```

6. Confirm the new cert is trusted inside the Ranger pod:

   ```bash
   kubectl -n ranger exec deploy/ranger-apache-ranger -- bash -c \
     '/opt/java/openjdk/bin/keytool -list -v \
        -keystore /opt/java/openjdk/jre/lib/security/cacerts \
        -storepass changeit -alias trino-ca' \
     | grep -E "Owner|Subject Alternative"
   # Expected: Owner: O=Trino, CN=trino.local
   ```

> **Note:** If you use cert-manager or another CA, replace steps 1-2 with your
> certificate workflow — the important part is that the ingress serves a cert whose SAN
> includes `trino.local`, and that same CA cert is in Ranger's `customCaCerts` secret.

### Fixes in 0.4.0

#### `PKIX path building failed` when connecting Ranger to Trino (or other TLS services)
**Cause:** Trino's TLS certificate is signed by a CA that is not in the JVM's default
truststore.

**Fix:** enable `customCaCerts` with a Secret containing the CA certificate — see
[Custom CA certificates](#custom-ca-certificates-tls-trust) above.

### Fixes in 0.3.0

#### Pod-level DNS / hostname resolution
**Fix:** added `hostAliases` support for injecting custom `/etc/hosts` entries into the
Ranger pod.

### Fixes in 0.2.0

These fix three stacked failures seen on a fresh install (notably on slow,
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
