# Changelog

All notable changes to this role will be documented in this file.

## [Unreleased]

### Added
- Initial implementation of `001-lab-metrics-backend` feature (US1–US4)
  - Deploy rootless InfluxDB 3 Core via Podman Quadlet (US1)
  - Named Podman volume backing with optional S3 (`s3fs-fuse`) support (US2)
  - Configurable data retention via `influxdb3_retention_days` (US3)
  - Export procedure documentation and Traefik integration (US4)
