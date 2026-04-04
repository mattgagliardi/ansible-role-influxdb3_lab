# metrics-aggregation — Ansible Role

Deploy and manage rootless **InfluxDB 3 Core** instances via **Podman Quadlet** on
Fedora 39+, RHEL 9.3+, and CentOS Stream 9.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Prerequisites](#2-prerequisites)
3. [Quick Start](#3-quick-start)
4. [Variable Reference](#4-variable-reference)
5. [Multiple Instances](#5-multiple-instances)
6. [S3-Backed Storage](#6-s3-backed-storage)
7. [Teardown](#7-teardown)
8. [Export Procedure](#8-export-procedure)
9. [Traefik Integration Notes](#9-traefik-integration-notes)
10. [Changelog](#10-changelog)

---

## 1. Purpose

This role deploys a single **InfluxDB 3 Core** container as a rootless Podman Quadlet
systemd user service. It is designed for lab and homelab environments where:

- Containers run as a non-root service user (never `root`)
- Persistent data lives in named Podman volumes (never bind mounts)
- Routing and TLS termination are handled by Traefik (labels-based discovery)
- Multiple isolated InfluxDB 3 instances can coexist on the same host

The role covers four user stories:
- **US1**: Deploy an InfluxDB 3 Core instance (container + Quadlet unit)
- **US2**: Persistent storage with optional S3 backing via `s3fs-fuse`
- **US3**: Configurable data retention period
- **US4**: Data export procedure documentation and Traefik integration notes

---

## 2. Prerequisites

### Host requirements

| Requirement | Minimum version | Notes |
|---|---|---|
| Ansible | 2.15 | Controller only |
| Podman | 5.0 | On the managed host |
| `containers.podman` collection | 1.5.3 | `ansible-galaxy collection install -r requirements.yml` |

### Managed host requirements

- Fedora 39+, RHEL 9.3+, or CentOS Stream 9
- A **pre-existing non-privileged service user** (default: `influxdb3`) with:
  - An entry in `/etc/subuid` and `/etc/subgid` (required for rootless containers)
  - Linger enabled (the role enables linger automatically via `loginctl`)
- `s3fs-fuse` package installed **only** if using S3-backed volumes

> ⚠️ **This role never creates or modifies OS user accounts.** The service user
> must exist before the role runs. The role enforces this via pre-flight assertion.

### Install the collection

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## 3. Quick Start

```yaml
# playbook.yml
- hosts: labhost01
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_instance_name: lab1
        influxdb3_hostname: lab1-influx.example.com
        influxdb3_admin_token: "{{ vault_lab1_token }}"   # ansible-vault encrypted
        influxdb3_retention_days: 30
```

```bash
ansible-playbook -i inventory playbook.yml --ask-vault-pass
```

After the play completes, InfluxDB 3 is available at `http://localhost:8181` (via the
loopback port binding) and routable via Traefik at `https://lab1-influx.example.com`.

---

## 4. Variable Reference

### Required variables (no default — role fails pre-flight if missing)

| Variable | Type | Description |
|---|---|---|
| `influxdb3_instance_name` | string | Unique instance identifier. Pattern: `^[a-z0-9][a-z0-9-]{1,28}[a-z0-9]$`. Example: `lab1`. |
| `influxdb3_hostname` | string | FQDN for the Traefik `Host()` routing rule. Must match your TLS certificate. |

### Optional variables

| Variable | Type | Default | Description |
|---|---|---|---|
| `influxdb3_image` | string | `docker.io/influxdb:3-core` | Container image. Use a digest-pinned reference in production. |
| `influxdb3_admin_token` | string | _(none)_ | Admin bearer token for the InfluxDB 3 API. Not used by the role at deploy time — InfluxDB 3 Core does not accept a pre-set token via env var. Create it post-deploy with `podman exec influxdb3-<instance> influxdb3 create token --admin`, then store it with `ansible-vault` and pass it here as a reference for downstream consumers (e.g. a Grafana datasource role). |
| `influxdb3_port` | integer | `8181` | HTTP API port. InfluxDB 3 changed the default from 8086 (v2) to **8181** (v3). |
| `influxdb3_shared_network_name` | string | `influxdb3-net` | Name of the Podman network all instances attach to. Set this to the name of a pre-existing network on the target host; the role will create it if it does not exist. All instances on the same host must use the same value. |
| `influxdb3_traefik_entrypoint` | string | `websecure` | Traefik entrypoint. Must match your TLS entrypoint config. |
| `influxdb3_retention_days` | integer | `90` | Default retention period in days (minimum: 1). |
| `influxdb3_volume_driver` | string | `local` | Podman volume driver. S3-backed storage uses `local` + `influxdb3_volume_options`. |
| `influxdb3_volume_options` | string | `""` | Comma-separated volume `Options=` string. Empty = plain local volume. |
| `influxdb3_service_user` | string | `influxdb3` | Pre-existing non-privileged user that owns rootless containers. |
| `influxdb3_quadlet_dir` | string | `~{{ influxdb3_service_user }}/.config/containers/systemd` | Quadlet unit file directory. |
| `influxdb3_env_file_path` | string | `~{{ influxdb3_service_user }}/.config/influxdb3/{{ influxdb3_instance_name }}.env` | Path to the environment file (mode 0600). |
| `influxdb3_teardown` | boolean | `false` | Set to `true` to remove the instance. Volumes are always preserved. |
| `influxdb3_startup_timeout_seconds` | integer | `120` | `TimeoutStartSec` and readiness probe timeout. |

---

## 5. Multiple Instances

Deploy two isolated instances on the same host by using separate plays with scoped
`vars:` blocks. Both instances share one Podman network (default: `influxdb3-net`, configurable via `influxdb3_shared_network_name`) but have
independent volumes, unit files, and environment files.

```yaml
# Play 1 — lab1
- hosts: labhost01
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_instance_name: lab1
        influxdb3_hostname: lab1-influx.example.com
        influxdb3_admin_token: "{{ vault_lab1_token }}"
        influxdb3_retention_days: 30

# Play 2 — lab2
- hosts: labhost01
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_instance_name: lab2
        influxdb3_hostname: lab2-influx.example.com
        influxdb3_admin_token: "{{ vault_lab2_token }}"
        influxdb3_retention_days: 90
```

---

## 6. S3-Backed Storage

Use the Podman `local` driver with `s3fs-fuse` to back the data volume with an S3 bucket.

### Prerequisites

- `s3fs-fuse` package installed on the managed host
- `~influxdb3/.passwd-s3fs` pre-configured with S3 credentials (the role does not
  read or write this file)
- The S3 bucket prefix (`/myprefix`) must already exist as an object key

### Example

```yaml
- hosts: labhost01
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_instance_name: lab2
        influxdb3_hostname: lab2-influx.example.com
        influxdb3_admin_token: "{{ vault_lab2_token }}"
        influxdb3_volume_options: "type=fuse.s3fs,device=mybucket:/lab2,o=use_xattr,endpoint=us-east-1,allow_other"
```

The role performs a pre-flight check that `s3fs` is available when
`influxdb3_volume_options` contains `fuse.s3fs`.

---

## 7. Teardown

To remove an instance, set `influxdb3_teardown: true`. This stops the service and
removes the Quadlet unit files and environment file.

> ⚠️ **Named Podman volumes are NEVER deleted by this role.** Your data is always
> preserved. Remove volumes manually with `podman volume rm <name>` if needed.

```yaml
- hosts: labhost01
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_instance_name: lab1
        influxdb3_hostname: lab1-influx.example.com
        influxdb3_admin_token: "{{ vault_lab1_token }}"
        influxdb3_teardown: true
```

---

## 8. Export Procedure

Export all databases to Parquet files using `podman exec` to run the `influxdb3` CLI
inside the running container.

```bash
# Variables — substitute the instance name used at deploy time
INSTANCE="mylab"
CONTAINER="influxdb3-${INSTANCE}"
TOKEN="<influxdb3_admin_token value>"
DEST="./influxdb3-${INSTANCE}-export-$(date +%Y%m%d-%H%M%S)"

# 1. Export all databases to a Parquet archive inside the container
podman exec --user influxdb "${CONTAINER}" \
  influxdb3 export \
    --token "${TOKEN}" \
    --output-dir /tmp/export

# 2. Copy the export directory from the container to the host
podman cp "${CONTAINER}:/tmp/export" "${DEST}"

# 3. Remove the temporary export directory inside the container
podman exec "${CONTAINER}" rm -rf /tmp/export

echo "Export written to: ${DEST}"
```

**Alternative (HTTP API)**:

```bash
curl -sf \
  -H "Authorization: Bearer ${TOKEN}" \
  "http://localhost:8181/api/v1/query?q=SELECT+*+FROM+<database>" \
  --output "${DEST}.csv"
```

**Artefact format**: Apache Parquet (columnar). Compatible with DuckDB, Polars,
PyArrow, Apache Spark, and InfluxDB 3 import. The export directory preserves the
database → table → partition hierarchy.

---

## 9. Traefik Integration Notes

This role emits **Traefik Docker-provider labels** on each container. Traefik must be
configured to watch the rootless Podman socket for the service user:

```
/run/user/<UID>/podman/podman.sock
```

Configure this in your Traefik provider config (not managed by this role).

**Labels emitted per instance** (router/service name: `influxdb3-<instance_name>`):

| Label | Value |
|---|---|
| `traefik.enable` | `true` |
| `traefik.http.routers.influxdb3-<name>.rule` | `Host(\`{{ influxdb3_hostname }}\`)` |
| `traefik.http.routers.influxdb3-<name>.entrypoints` | `{{ influxdb3_traefik_entrypoint }}` |
| `traefik.http.routers.influxdb3-<name>.tls` | `true` |
| `traefik.http.services.influxdb3-<name>.loadbalancer.server.port` | `8181` |

> ⚠️ The `loadbalancer.server.port` is always **8181** (InfluxDB 3 default).
> Do not use 8086 — that is the InfluxDB 2.x port.

The port binding `PublishPort=127.0.0.1:8181:8181` exposes the API on the host
loopback for local health checks. Traefik routes to the container by IP on the
`influxdb3-net` Podman network, not via the published port.

---

## 10. Changelog

See [CHANGELOG.md](CHANGELOG.md).
