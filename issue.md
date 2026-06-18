# Apache Ranger Chart — Issues Found & Fixes Applied

This document summarizes the debugging and fixes applied to the `apache-ranger` Helm
chart. The original release (`apache-ranger-0.1.1`, appVersion 2.7.0) failed to start
with a PostgreSQL error. Investigation revealed **three stacked bugs**; all were fixed
and the chart was re-released as **`0.2.0`** (appVersion 2.8.0), plus an Ingress was
added.

- **Namespace:** `ranger`
- **Result:** Clean install now comes up end-to-end with no manual steps. Verified via
  `ranger.local` ingress returning HTTP `200`, and the authenticated REST API
  (`/service/plugins/services`) returning `200`.

---

## Issue 1 — PostgreSQL: `password authentication failed for user "postgres"`

### Symptom
The Ranger pod looped forever on `PostgreSQL not ready, retrying in 5s...`. PostgreSQL
logs showed:

```
FATAL:  password authentication failed for user "postgres"
DETAIL:  User "postgres" has no password assigned.
```

### Root cause
On the cluster's slow NFS storage, PostgreSQL `initdb` takes ~4+ minutes. The Bitnami
subchart's **default `livenessProbe`** (initialDelay 30s + ~6×10s) kills the container
~90–120s after start. The first container was **SIGKILLed (exit 137) mid-initialization**
— after `initdb` created the cluster but **before** the Bitnami entrypoint ran
"Changing password of postgres / Creating user ranger". On restart, the entrypoint saw a
half-written data directory ("Deploying PostgreSQL with persisted data...") and
**permanently skipped** credential setup, leaving `postgres` with a NULL password.

### Fix (`values.yaml` + `ranger-values.yaml`)
Enable a generous `startupProbe` on the PostgreSQL primary so liveness/readiness are
gated until init completes, and relax the liveness probe:

```yaml
postgresql:
  primary:
    startupProbe:
      enabled: true
      initialDelaySeconds: 30
      periodSeconds: 15
      timeoutSeconds: 5
      failureThreshold: 40   # ~30s + 40*15s ≈ 10 min budget for first-time init
    livenessProbe:
      enabled: true
      initialDelaySeconds: 60
      periodSeconds: 10
      timeoutSeconds: 5
      failureThreshold: 6
```

> Recovery for an already-corrupted volume: delete the PVC so the DB reinitializes:
> `kubectl delete pvc data-ranger-postgresql-0 -n ranger && kubectl delete pod ranger-postgresql-0 -n ranger`

---

## Issue 2 — Ranger connects to `ranger-db` with an empty DB user

### Symptom
Once PostgreSQL was reachable, Ranger setup failed with:

```
[JISQL] ... -cstring jdbc:postgresql://ranger-db/ranger -u  -p '********' ...
SQLException : SQL state: 08001 The connection attempt failed.
```

Note the host `ranger-db`, database `ranger`, and **empty user** — none match the chart's
configured `ranger-postgresql` / `rangerdb` / `ranger`.

### Root cause
The chart mounts a correct `install.properties`, but the image entrypoint
`/home/ranger/scripts/ranger.sh` **overwrites** it with the image's default
`ranger-admin-install.properties` (`db_host=ranger-db`, `db_name=ranger`) and only appends
DB credentials from env vars (`RANGER_DB_USER`, `RANGER_DB_PASSWORD`, `POSTGRES_PASSWORD`)
that **the chart never sets** — hence the wrong host and empty user.

### Fix (`templates/deployment.yaml`)
Do **not** run `ranger.sh`. After waiting for PostgreSQL, drive `setup.sh` directly
against the chart-provided `install.properties` (correct host/db/user), start the admin
service, and keep the container alive on the admin PID:

```bash
cd /opt/ranger/admin
if [ ! -e /opt/ranger/.setupDone ]; then
  ./setup.sh && touch /opt/ranger/.setupDone
fi
./ews/ranger-admin-services.sh start
sleep 30
python3 /home/ranger/scripts/create-ranger-services.py || true
RANGER_ADMIN_PID=$(ps -ef | grep -v grep | grep -i "org.apache.ranger.server.tomcat.EmbeddedServer" | awk '{print $2}')
tail --pid=$RANGER_ADMIN_PID -f /dev/null
```

Result: Ranger connects to `jdbc:postgresql://ranger-postgresql/rangerdb -u ranger` and
imports its schema (~78 tables).

---

## Issue 3 — `NumberFormatException` on Ranger Admin startup

### Symptom
Tomcat started but the webapp context failed:

```
SEVERE: Context [] startup failed due to previous errors
java.lang.NumberFormatException: For input string:
  "...[E] 'token_valid' not found in /opt/ranger/admin/install.properties file while getting....!!"
```

The generated `ranger-admin-site.xml` had:

```xml
<name>ranger.admin.kerberos.token.valid.seconds</name>
<value>... [E] 'token_valid' not found ...!!</value>
```

### Root cause
`setup.sh`'s property getter, when a key is **absent**, returns an *error string* (which
is non-empty). `setup.sh` then writes that error string into the config (e.g.
`ranger.admin.kerberos.token.valid.seconds`), which fails numeric parsing at startup. The
chart's `install.properties` was missing 15 keys that the image's default template defines
(Kerberos/SPNEGO principals & keytabs, `token_valid`, and audit-JAAS options).

### Fix (`templates/configmap.yaml`)
Add every missing key (empty, matching the image default). When the key is present-but-
empty, `setup.sh` keeps the shipped default (e.g. `token.valid.seconds = 30`):

```
spnego_principal=
spnego_keytab=
token_valid=
admin_principal=
admin_keytab=
lookup_principal=
lookup_keytab=
audit_jaas_client_loginModuleName=
audit_jaas_client_loginModuleControlFlag=
audit_jaas_client_option_useKeyTab=
audit_jaas_client_option_storeKey=
audit_jaas_client_option_useTicketCache=
audit_jaas_client_option_serviceName=
audit_jaas_client_option_keyTab=
audit_jaas_client_option_principal=
```

---

## Additional changes

### Ingress (NGINX, host `ranger.local`)
Added an Ingress (template already existed) and configured it in `ranger-values.yaml`:

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
```

Point `ranger.local` at the ingress controller address (node `192.168.2.100` for the
RKE2 hostNetwork ingress-nginx): add `192.168.2.100  ranger.local` to `/etc/hosts`.

### Config checksum auto-rollout (`templates/deployment.yaml`)
Added a `checksum/config` pod annotation so ConfigMap changes automatically roll the
Deployment (no manual `kubectl rollout restart` needed).

### Version, app version & image
- `Chart.yaml`: `version 0.1.1 → 0.2.0`, `appVersion 2.7.0 → 2.8.0`.
- `image.tag`: `2.7.0 → 2.8.0`.
- PostgreSQL image kept at `bitnamilegacy/postgresql:14.17.0-debian-12-r3` (Bitnami moved
  its free images to the `bitnamilegacy` namespace; `bitnami/postgresql:16.x` no longer
  pulls).

### Useful NOTES.txt
Replaced the placeholder NOTES with real post-install instructions (access URL/port-
forward, credentials, startup-time warning).

### Chart maintainer
Set the chart owner/maintainer in `Chart.yaml` to **danghs198** (`danghs198@gmail.com`).

---

## Files changed

| File | Change |
| --- | --- |
| `apache-ranger/Chart.yaml` | version 0.2.0, appVersion 2.8.0, maintainer danghs198 |
| `apache-ranger/values.yaml` | startupProbe/livenessProbe, image tag 2.8.0, ingress block |
| `apache-ranger/templates/deployment.yaml` | bypass `ranger.sh`, run `setup.sh` directly, config checksum annotation |
| `apache-ranger/templates/configmap.yaml` | added 15 missing properties |
| `apache-ranger/templates/NOTES.txt` | real access/credential instructions |
| `ranger-values.yaml` | startupProbe/livenessProbe, ingress enabled (ranger.local) |
| `apache-ranger-0.2.0.tgz` | newly packaged chart |
| `README.md` | chart documentation |

---

## Verification

```
$ kubectl get pods -n ranger
NAME                                    READY   STATUS    RESTARTS   AGE
ranger-apache-ranger-...                1/1     Running   0          ...
ranger-postgresql-0                     1/1     Running   0          ...

$ curl -H "Host: ranger.local" http://192.168.2.100/login.jsp        -> HTTP 200
$ curl -u admin:StrongPass58 .../service/plugins/services            -> HTTP 200
```

A full clean cycle (uninstall + PVC delete + fresh `helm install`) was performed and the
release came up automatically with no manual intervention. Default admin credentials:
`admin` / `StrongPass58`.
