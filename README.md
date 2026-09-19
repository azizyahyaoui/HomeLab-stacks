# Homelab Stack

Infrastructure and services for a self-hosted homelab. Each top-level directory is
kept independent so services can be deployed, updated, and backed up separately.

## Repository layout

| Directory | Purpose | Status |
| --- | --- | --- |
| [`apps/`](apps/) | Application stacks | Planned |
| [`containers/`](containers/) | General-purpose container definitions | Planned |
| [`databases/`](databases/) | Database services and persistent data definitions | Planned |
| [`homepage/`](homepage/) | Homepage dashboard and service links | Active |
| [`monitoring/`](monitoring/) | Metrics, logs, and observability | In progress |
| [`reverse-proxy/`](reverse-proxy/) | Ingress and TLS configuration | Planned |
| [`security/`](security/) | Vulnerability scanning and security tooling | In progress |

## Current services

- [Homepage](homepage/) — dashboard for homelab services and integrations.
- [ELK stack](monitoring/elk-stack/) — Elasticsearch, Logstash, and Kibana for log collection and analysis.
- [Belk](monitoring/Belk/) — monitoring stack workspace.
- [Prometheus](monitoring/promethus/) — metrics monitoring workspace.
- [Nessus](security/nessus/) — vulnerability scanning workspace.
- [OpenVAS](security/openvas/) — vulnerability scanning workspace.

## Getting started

### Prerequisites

- Docker Engine
- Docker Compose v2 (`docker compose`)
- A Linux host with enough storage for persistent service data

### Start a service

Move into the service directory and follow its local README. For a Compose-based
service, the usual workflow is:

```bash
cd path/to/service
docker compose up -d
docker compose ps
docker compose logs -f
```

Stop a service with:

```bash
docker compose down
```

Do not run Compose from the repository root unless a root-level compose file is
added in the future.

## Configuration and secrets

- Keep service-specific configuration beside the service that uses it.
- Commit sanitized `.env.sample` files, never populated `.env` files.
- Do not commit passwords, API keys, certificates, private keys, or generated data.
- Back up persistent volumes and document restore steps in the relevant service README.

## Contributing

When adding a service:

1. Create a dedicated directory under the most relevant top-level category.
2. Add a README with purpose, prerequisites, ports, volumes, configuration, and recovery steps.
3. Include a sanitized `.env.sample` when environment variables are required.
4. Record the service and its deployment instructions in this README.

## Roadmap

- Define shared network and storage conventions.
- Add reverse-proxy and TLS documentation.
- Document backup and restore procedures.
- Add health checks and monitoring targets for deployed services.
