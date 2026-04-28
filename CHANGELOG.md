# Changelog

All notable changes to this role will be documented in this file.

## [Unreleased]

### Changed

- **Default authentication behavior:** the role now starts InfluxDB 3 with
  `--without-auth` by default so unauthenticated clients (e.g. Telegraf) work
  out of the box. Previously the role rendered `--disable-authz health,ping`,
  which left the auth subsystem active for write/query endpoints — combined
  with the role not generating a token, this made default deployments
  effectively unusable for ingest. Operators who want auth must now set
  `influxdb3_auth_enabled: true` and generate the admin token post-deploy
  with `podman exec influxdb3-<instance> influxdb3 create token --admin`.

### Added

- `influxdb3_auth_enabled` (boolean, default `false`) toggles the auth
  subsystem. See README §6 "Authentication".
- Initial implementation of `001-lab-metrics-backend` feature (US1–US4)
  - Deploy rootless InfluxDB 3 Core via Podman Quadlet (US1)
  - Named Podman volume backing with optional S3 (`s3fs-fuse`) support (US2)
  - Configurable data retention via `influxdb3_retention_days` (US3)
  - Export procedure documentation and Traefik integration (US4)
