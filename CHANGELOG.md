# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
[markdownlint](https://dlaa.me/markdownlint/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Initial general performance and diagnostics FAQ for SENZING® v4 software.
- Guidance on collecting `SzEngine.getStats()` workload JSON, database wait statistics and host
  telemetry for Senzing Support.
- Links to the backend-specific companion repositories (`mssql-performance-v4`,
  `postgresql-performance-v4`).

### Changed

- Diagnostics no longer rely on symbolized stack traces: Senzing v4 software ships stripped libraries with no
  build ID, so the previous "attach a debugger and read the Senzing call stack" technique has been
  replaced with engine workload counters, database wait/contention statistics and host telemetry.
