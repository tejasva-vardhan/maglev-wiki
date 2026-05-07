**Status:** Proposed
 
**Owner:** Aaron Brethorst
 
## Context
 
Maglev is currently configured via one of two mechanisms: a JSON file passed with the `-f` flag, or CLI flags. Neither approach follows [12-factor app principle III](https://12factor.net/config), which calls for configuration to be stored in the environment.
 
The JSON file is the intended production mechanism. A small number of environment variables (`GTFS_API_KEYS`, `MAGLEV_LOG_LEVEL`, and a few others) exist as ad-hoc overrides bolted on top, but they cover only a fraction of available settings. Most fields — port, GTFS feed URLs, data path, TLS paths, rate limit — cannot be set via environment variable at all. The realtime auth header overrides only apply to the first realtime feed.
 
This creates several practical problems:
 
- Secrets such as API keys and auth tokens must live in `config.json`, which has to be excluded from source control by hand.
- Container deployments (Docker, ECS, Render, Kubernetes) cannot be configured via env vars alone; a config file must be injected at runtime, which is awkward on platforms where env vars are first-class citizens.
- The override list grows ad hoc as new needs arise, with no systematic rule operators can rely on.
We are approaching 1.0 and want to stabilize the configuration surface before committing to backwards compatibility. The direction has been discussed among active contributors and is uncontroversial; the question is how to layer the file and the environment cleanly. SSL/TLS termination came up in discussion and is explicitly out of scope here — Maglev will continue to assume operators terminate TLS upstream (managed load balancer, PaaS, or a reverse proxy such as Caddy or Nginx). A separate ADR can revisit this if needed.
 
## Decision
 
We will treat `config.json` as checked-in defaults and environment variables as overrides applied on top of it, using a hierarchical mapping scheme modelled on the [.NET configuration provider](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/configuration).
 
**Mapping rule.** A JSON key path becomes an environment variable name by:
 
1. Prefixing with `MAGLEV`
2. Joining path segments with double underscores `__`
3. Converting hyphens to underscores within each segment
4. Upper-casing the result
Examples:
 
| JSON path                                | Environment variable                              |
|------------------------------------------|---------------------------------------------------|
| `port`                                   | `MAGLEV__PORT`                                    |
| `env`                                    | `MAGLEV__ENV`                                     |
| `rate-limit`                             | `MAGLEV__RATE_LIMIT`                              |
| `log-level`                              | `MAGLEV__LOG_LEVEL`                               |
| `gtfs-static-feed.url`                   | `MAGLEV__GTFS_STATIC_FEED__URL`                   |
| `gtfs-static-feed.auth-header-value`     | `MAGLEV__GTFS_STATIC_FEED__AUTH_HEADER_VALUE`     |
| `gtfs-rt-feeds[0].trip-updates-url`      | `MAGLEV__GTFS_RT_FEEDS__0__TRIP_UPDATES_URL`      |
| `gtfs-rt-feeds[1].vehicle-positions-url` | `MAGLEV__GTFS_RT_FEEDS__1__VEHICLE_POSITIONS_URL` |
| `data-path`                              | `MAGLEV__DATA_PATH`                               |
| `tls-cert-path`                          | `MAGLEV__TLS_CERT_PATH`                           |
 
**Simple string arrays.** For `api-keys`, `protected-api-keys`, and `exempt-api-keys`, we will retain comma-separated parsing because it is more ergonomic than indexed variables:
 
```
MAGLEV__API_KEYS=key-one,key-two,key-three
```
 
The indexed form (`MAGLEV__API_KEYS__0=key-one`) is not supported for these fields; the comma-separated form is the only env var path.
 
**Config file as checked-in defaults.** `config.json` will be committed to the repository containing non-secret default values (URLs, port, log level, and so on). Placeholder values will be used for any field that must hold a secret in production:
 
```json
{
  "api-keys": ["change-me"],
  "gtfs-static-feed": {
    "auth-header-value": ""
  }
}
```
 
Secrets are never stored in the file; they are injected via env vars at deploy time.
 
**Precedence order** (highest to lowest):
 
1. Environment variables (`MAGLEV__*`)
2. JSON config file (`-f` flag, or a default path if present)
3. Built-in defaults (applied by `setDefaults()`)
**Deprecation of legacy env vars.** The following existing variables will continue to work in a compatibility shim for one release cycle, then be removed:
 
| Legacy variable            | Replacement                                            |
|----------------------------|--------------------------------------------------------|
| `GTFS_API_KEYS`            | `MAGLEV__API_KEYS`                                     |
| `GTFS_PROTECTED_API_KEYS`  | `MAGLEV__PROTECTED_API_KEYS`                           |
| `MAGLEV_LOG_LEVEL`         | `MAGLEV__LOG_LEVEL`                                    |
| `MAGLEV_LOG_FORMAT`        | `MAGLEV__LOG_FORMAT`                                   |
| `GTFS_STATIC_AUTH_NAME`    | `MAGLEV__GTFS_STATIC_FEED__AUTH_HEADER_NAME`           |
| `GTFS_STATIC_AUTH_VALUE`   | `MAGLEV__GTFS_STATIC_FEED__AUTH_HEADER_VALUE`          |
| `GTFS_REALTIME_AUTH_NAME`  | `MAGLEV__GTFS_RT_FEEDS__0__REALTIME_AUTH_HEADER_NAME`  |
| `GTFS_REALTIME_AUTH_VALUE` | `MAGLEV__GTFS_RT_FEEDS__0__REALTIME_AUTH_HEADER_VALUE` |
 
The configuration surface — field names, types, and the env-var-to-field mapping — is considered stable as of 1.0. Changes after 1.0 follow our normal compatibility policy.
 
### Alternatives considered
 
*Pure env vars, no config file.* Require all configuration via env vars only, removing the JSON file path. Rejected because complex nested config (multiple RT feeds with headers, agency ID lists) becomes unmanageable as a flat list of env vars, and the file provides a valuable self-documenting defaults artifact.
 
*Keep the current ad-hoc env var list.* Continue adding individual `os.Getenv` calls as new overrides are needed. Rejected because it does not scale, is undiscoverable, and leaves most settings unreachable from the environment.
 
## Consequences
 
**Positive**
 
- Every config field is overridable via env var. The rule is derivable, so operators do not need to consult a list of supported variables.
- Secrets can be injected at runtime without a config file present, making container and PaaS deployments straightforward.
- The checked-in `config.json` serves as self-documenting defaults and keeps local development simple — clone the repo and run.
- The configuration surface is locked down before 1.0, reducing the risk of breaking changes shortly after release.
- Aligns Maglev with 12-factor expectations that most operators already hold.

**Negative**
 
- Array-indexed env vars for `gtfs-rt-feeds` are verbose. Operators configuring a second feed entirely via env vars must set many variables; the file remains the more ergonomic path for complex feed configuration.
- Hyphen-to-underscore normalization means `gtfs-static-feed` and a hypothetical `gtfs_static_feed` key would collide. This is not a concern today because the JSON schema uses hyphens consistently, but it constrains future field naming.
- Implementation complexity: the override layer must walk the `JSONConfig` struct reflectively or via an explicit mapping table, apply type coercions, and validate after merging.
- Two configuration mechanisms must be documented and tested. Layering rules ("env beats file beats defaults") must be communicated clearly to avoid surprise. A startup log line that prints the resolved configuration would help operators audit the effective values.