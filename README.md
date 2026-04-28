# metrics-aggregation — Ansible Role

Deploy and manage rootless **InfluxDB 3 Core** instances via **Podman Quadlet** on
Fedora 39+, RHEL 9.3+, and CentOS Stream 9.

---

## Table of Contents

1. [Purpose](#1-purpose)
2. [Prerequisites](#2-prerequisites)
3. [Quick Start](#3-quick-start)
4. [Variable Reference](#4-variable-reference)
5. [Shared Playbook Variables](#5-shared-playbook-variables)
6. [Authentication](#6-authentication)
7. [Multiple Instances](#7-multiple-instances)
8. [S3-Backed Storage](#8-s3-backed-storage)
9. [Teardown](#9-teardown)
10. [Export Procedure](#10-export-procedure)
11. [Traefik Integration Notes](#11-traefik-integration-notes)
12. [Changelog](#12-changelog)

---

## 1. Purpose

This role deploys a single **InfluxDB 3 Core** container as a rootless Podman Quadlet
systemd user service. It is designed for lab and homelab environments where:

- Containers run as the Ansible connecting user (no dedicated service account required)
- Persistent data lives in Quadlet-managed volumes (`.volume` unit files)
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

| Requirement                    | Minimum version | Notes                                                   |
| ------------------------------ | --------------- | ------------------------------------------------------- |
| Ansible                        | 2.15            | Controller only                                         |
| Podman                         | 5.0             | On the managed host                                     |
| `containers.podman` collection | 1.5.3           | `ansible-galaxy collection install -r requirements.yml` |

### Managed host requirements

- Fedora 39+, RHEL 9.3+, or CentOS Stream 9
- Rootless Podman configured for the connecting user (linger enabled automatically by role)
- `s3fs-fuse` package installed **only** if using S3-backed volumes

> ℹ️ **No dedicated service account is needed.** This role runs as `ansible_user_id`
> (the user Ansible connects as). It will refuse to run as `root` unless
> `influxdb3_allow_root_for_testing: true` is explicitly set (CI use only).

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
        influxdb3_traefik_domain: example.internal
        influxdb3_retention_days: 30
```

```bash
ansible-playbook -i inventory playbook.yml --ask-vault-pass
```

After the play completes, InfluxDB 3 is available at
`http://localhost:{{ influxdb3_http_port }}` (via the loopback port binding) and
routable via Traefik at `https://lab1.influxdb3.example.internal`.

---

## 4. Variable Reference

### Required variables (no default — role fails pre-flight if missing)

| Variable                   | Type   | Description                                                                                                                          |
| -------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| `influxdb3_instance_name`  | string | Unique instance identifier. Pattern: `^[a-z0-9][a-z0-9-]{1,28}[a-z0-9]$`. Example: `lab1`. Falls back to `lab_instance_name` if set. |
| `influxdb3_traefik_domain` | string | Domain used to build the Traefik routing rule: `Host(\`<instance>.influxdb3.<domain>\`)`. Falls back to `lab_traefik_domain` if set. |

### Optional variables

| Variable                            | Type    | Default                                                                      | Description                                                                                                                                                                                                                                                                                                      |
| ----------------------------------- | ------- | ---------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `influxdb3_image`                   | string  | `docker.io/influxdb:3-core`                                                  | Container image. Use a digest-pinned reference in production.                                                                                                                                                                                                                                                    |
| `influxdb3_auth_enabled`            | boolean | `false`                                                                      | When `false` (default), InfluxDB 3 starts with `--without-auth` and accepts unauthenticated requests on every endpoint. When `true`, auth is enforced for write/query endpoints; health and `/ping` remain unauthenticated so probes work. The role does **not** generate a token — see `influxdb3_admin_token`. |
| `influxdb3_admin_token`             | string  | _(none)_                                                                     | Operator-side bearer token. **Not consumed by this role.** Only relevant when `influxdb3_auth_enabled: true`; generate post-deploy with `podman exec influxdb3-<instance> influxdb3 create token --admin` and store with `ansible-vault` for use by clients (Telegraf, export scripts).                          |
| `influxdb3_http_port`               | integer | `8181`                                                                       | HTTP API port. InfluxDB 3 changed the default from 8086 (v2) to **8181** (v3).                                                                                                                                                                                                                                   |
| `influxdb3_shared_network_name`     | string  | `influxdb3-net`                                                              | Name of the Podman network all instances attach to. Created if it does not exist. All instances on the same host must use the same value.                                                                                                                                                                        |
| `influxdb3_traefik_entrypoint`      | string  | `websecure`                                                                  | Traefik entrypoint. Must match your TLS entrypoint config. Falls back to `lab_traefik_entrypoint` if set.                                                                                                                                                                                                        |
| `influxdb3_retention_days`          | integer | `90`                                                                         | Default retention period in days (minimum: 1).                                                                                                                                                                                                                                                                   |
| `influxdb3_storage_backend`         | string  | `local`                                                                      | Storage backend: `local` (Podman-managed volume) or `s3` (s3fs-fuse bind-mount via `.volume` unit).                                                                                                                                                                                                              |
| `influxdb3_s3_mount_path`           | string  | _(none)_                                                                     | Absolute path to a live s3fs-fuse mountpoint on the managed host. Required when `influxdb3_storage_backend: s3`.                                                                                                                                                                                                 |
| `influxdb3_quadlet_dir`             | string  | `{{ ansible_user_dir }}/.config/containers/systemd`                          | Quadlet unit file directory.                                                                                                                                                                                                                                                                                     |
| `influxdb3_env_file_path`           | string  | `{{ ansible_user_dir }}/.config/influxdb3/{{ influxdb3_instance_name }}.env` | Path to the environment file (mode 0600).                                                                                                                                                                                                                                                                        |
| `influxdb3_teardown`                | boolean | `false`                                                                      | Set to `true` to remove the instance. Volumes are always preserved.                                                                                                                                                                                                                                              |
| `influxdb3_startup_timeout_seconds` | integer | `120`                                                                        | `TimeoutStartSec` and readiness probe timeout.                                                                                                                                                                                                                                                                   |
| `influxdb3_allow_root_for_testing`  | boolean | `false`                                                                      | CI escape hatch. Allows the role to run as root. Never set in production.                                                                                                                                                                                                                                        |

---

## 5. Shared Playbook Variables

When calling both `metrics-aggregation` and `loki_lab` from the same playbook, you can
set shared values once at the play level using the `lab_*` convention:

| Shared variable          | Used by                                                   |
| ------------------------ | --------------------------------------------------------- |
| `lab_instance_name`      | `influxdb3_instance_name`, `loki_instance_name`           |
| `lab_traefik_domain`     | `influxdb3_traefik_domain`, `loki_traefik_domain`         |
| `lab_traefik_entrypoint` | `influxdb3_traefik_entrypoint`, `loki_traefik_entrypoint` |

```yaml
- hosts: labhost01
  vars:
    lab_instance_name: lab1
    lab_traefik_domain: example.internal
    lab_traefik_entrypoint: websecure
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_admin_token: "{{ vault_lab1_influxdb_token }}"
    - role: loki_lab
```

---

## 6. Authentication

By default this role starts InfluxDB 3 with `--without-auth`. All endpoints
accept unauthenticated requests, which keeps shipping data in (e.g. from
Telegraf) friction-free for lab and homelab deployments.

To enable authentication, set `influxdb3_auth_enabled: true`:

```yaml
- hosts: labhost01
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_instance_name: lab1
        influxdb3_traefik_domain: example.internal
        influxdb3_auth_enabled: true
```

The role does **not** generate the admin token. After deploy, generate one
with:

```bash
podman exec influxdb3-lab1 influxdb3 create token --admin
```

Store the resulting token with `ansible-vault` (for example as
`vault_lab1_token`) and reference it as `influxdb3_admin_token` in any
playbooks or scripts that need to authenticate against the API. Health and
`/ping` remain unauthenticated regardless, so monitoring probes continue to
work without a token.

---

## 7. Multiple Instances

Deploy two isolated instances on the same host by using separate plays with scoped
`vars:` blocks. Both instances share one Podman network (default: `influxdb3-net`,
configurable via `influxdb3_shared_network_name`) but have independent volumes, unit
files, and environment files.

```yaml
# Play 1 — lab1
- hosts: labhost01
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_instance_name: lab1
        influxdb3_traefik_domain: example.internal
        influxdb3_admin_token: "{{ vault_lab1_token }}"
        influxdb3_retention_days: 30

# Play 2 — lab2
- hosts: labhost01
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_instance_name: lab2
        influxdb3_traefik_domain: example.internal
        influxdb3_http_port: 8182
        influxdb3_admin_token: "{{ vault_lab2_token }}"
        influxdb3_retention_days: 90
```

---

## 8. S3-Backed Storage

Use `influxdb3_storage_backend: s3` to back the data volume with an S3 bucket via
`s3fs-fuse`. The role renders a Quadlet `.volume` unit file that bind-mounts the
s3fs-fuse mountpoint into the container.

### Prerequisites

- `s3fs-fuse` package installed and configured on the managed host
- The s3fs-fuse mountpoint (`influxdb3_s3_mount_path`) must be live before the role runs
- The role does not manage S3 credentials or bucket creation

### Example

```yaml
- hosts: labhost01
  roles:
    - role: metrics-aggregation
      vars:
        influxdb3_instance_name: lab1
        influxdb3_traefik_domain: example.internal
        influxdb3_admin_token: "{{ vault_lab1_token }}"
        influxdb3_storage_backend: s3
        influxdb3_s3_mount_path: /mnt/s3/influxdb3-lab1
```

The role performs a pre-flight check that `influxdb3_s3_mount_path` is defined and
is an active mountpoint when `influxdb3_storage_backend: s3`.

---

## 9. Teardown

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
        influxdb3_traefik_domain: example.internal
        influxdb3_teardown: true
```

---

## 10. Export Procedure

Export all databases to Parquet files using `podman exec` to run the `influxdb3` CLI
inside the running container.

```bash
# Variables — substitute the instance name used at deploy time
INSTANCE="lab1"
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
PyArrow, Apache Spark, and InfluxDB 3 import.

---

## 11. Traefik Integration Notes

This role emits **Traefik Docker-provider labels** on each container. Traefik must be
configured to watch the rootless Podman socket for the connecting user:

```
/run/user/<UID>/podman/podman.sock
```

Configure this in your Traefik provider config (not managed by this role).

**Labels emitted per instance** (router/service name: `influxdb3-<instance_name>`):

| Label                                                             | Value                                     |
| ----------------------------------------------------------------- | ----------------------------------------- |
| `traefik.enable`                                                  | `true`                                    |
| `traefik.http.routers.influxdb3-<name>.rule`                      | `Host(\`<instance>.influxdb3.<domain>\`)` |
| `traefik.http.routers.influxdb3-<name>.entrypoints`               | `{{ influxdb3_traefik_entrypoint }}`      |
| `traefik.http.routers.influxdb3-<name>.tls`                       | `true`                                    |
| `traefik.http.services.influxdb3-<name>.loadbalancer.server.port` | `{{ influxdb3_http_port }}`               |

The port binding `PublishPort=127.0.0.1:{{ influxdb3_http_port }}:{{ influxdb3_http_port }}`
exposes the API on the host loopback for local health checks. Traefik routes to the
container by IP on the `influxdb3-net` Podman network, not via the published port.

---

## 12. Changelog

See [CHANGELOG.md](CHANGELOG.md).

Test commit.
