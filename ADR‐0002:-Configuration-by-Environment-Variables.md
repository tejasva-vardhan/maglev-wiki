# ADR-0002: Configuration via Environment Variables

**Status:** Proposed

**Owner:** Aaron Brethorst

## Context

Maglev's runtime configuration is currently driven primarily by a `config.json` file. A handful of fields — API keys, log levels — can be overridden via environment variables, but most cannot. Notably, critical fields like GTFS feed URLs have no environment variable equivalent.

This creates friction for cloud deployments. On platforms like AWS ECS, Render, Fly, and Kubernetes, environment variables are first-class citizens: they are easy to set in task definitions, manifests, and dashboards, and they integrate cleanly with secret managers. Configuration files, by contrast, must be baked into images or mounted from external volumes or secret stores, which adds operational complexity for every deployment.

The [[12-factor app](https://12factor.net/config)](https://12factor.net/config) guidance treats environment variables as the canonical mechanism for environment-specific configuration, and most operators of Maglev in cloud environments already expect this. Local development, however, benefits from a checked-in file: it is easier to read, diff, and reason about than a long list of `export` statements, and it avoids forcing every contributor to maintain a personal `.env`.

We are approaching a 1.0 release and want to stabilize the configuration surface before committing to backwards compatibility. Several active contributors have weighed in and the direction is uncontroversial; the open questions are about layering and scope.

A related question — how Maglev should handle SSL/TLS termination — surfaced in discussion. That concern is out of scope for this ADR and will be addressed separately.

## Decision

We will treat `config.json` as the base configuration and environment variables as overrides applied on top of it.

Specifically:

1. Every meaningful configuration field in Maglev will have a corresponding environment variable. When set, the environment variable takes precedence over the value in `config.json`.
2. A `config.json` with safe placeholder values will be checked into the repository so that local development works out of the box without requiring contributors to assemble configuration from scratch.
3. Nested or list-valued fields (such as the GTFS feeds array) will use a documented naming and encoding convention for their environment variable equivalents. The exact convention will be settled in the implementation issue, but the principle is that no field is reachable only via the file.
4. The configuration surface — field names, types, and the env-var-to-field mapping — will be considered stable as of 1.0. Changes after 1.0 follow our normal compatibility policy.
5. SSL/TLS termination is explicitly out of scope. Maglev will continue to assume that operators terminate TLS upstream (via a managed load balancer, PaaS, or a reverse proxy such as Caddy or Nginx). A separate ADR will revisit this if needed.

## Consequences

**Positive**

- Cloud deployments become straightforward: an ECS task definition, Render service, or Kubernetes manifest can configure Maglev entirely through environment variables, with no mounted config file required.
- Secrets (API keys, database credentials) flow naturally through platform secret managers without special handling.
- Local development remains simple: clone the repo, run the server, and the checked-in `config.json` provides working defaults.
- Aligns Maglev with 12-factor expectations that most operators already hold.
- Locks down the config surface before 1.0, reducing the risk of breaking changes shortly after release.

**Negative**

- Two configuration mechanisms must be maintained, documented, and tested. Layering rules ("env beats file") need to be clearly communicated to avoid surprise.
- Nested and list-valued fields require a naming convention that is inevitably less ergonomic than JSON. Operators configuring multiple GTFS feeds via env vars will have a slightly awkward experience compared to editing a file.
- A checked-in `config.json` invites contributors to commit local edits by accident. We will need a `.gitignore` strategy or a `config.example.json` pattern to mitigate this; the implementation issue will resolve which.
- Auditing the effective configuration at runtime becomes slightly harder, since the running values are a merge of two sources. We may want a startup log line or a debug endpoint that prints the resolved configuration.