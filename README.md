# Homelab Stack

Creator: Yahyaoui Med Aziz | 260727
Editor: Antigravity

[![HOMELAB](https://img.shields.io/badge/HOMELAB-STACK-0b84c6?style=flat-square)](.)
[![DOCKER](https://img.shields.io/badge/DOCKER-CONTAINERS-2496ed?style=flat-square&logo=docker&logoColor=white)](.)
[![MONITORING](https://img.shields.io/badge/MONITORING-OBSERVABILITY-6f42c1?style=flat-square)](monitoring/)
[![SECURITY OPERATIONS](https://img.shields.io/badge/SECURITY%20OPERATIONS-ACTIVE-d64545?style=flat-square)](security/)
[![VULNERABILITY SCANNING](https://img.shields.io/badge/VULNERABILITY%20SCANNING-IN%20PROGRESS-f08c46?style=flat-square)](security/)
[![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-MAPPED-4caf00?style=flat-square)](security/)

This repository contains a modular self-hosted homelab environment for running dashboards, observability, security tools, and supporting infrastructure as independent stacks. Each top-level directory is designed to be deployed, updated, and backed up separately so the whole lab remains manageable without coupling everything to a single compose file.

## Overview

The stack is structured around practical homelab needs:

- A central dashboard for service access and navigation
- Log collection and analysis for troubleshooting and auditing
- Monitoring and alerting for infrastructure health
- Vulnerability scanning and telemetry collection
- Infrastructure automation for VMs and lab provisioning

## Repository layout

| Directory | Purpose | Status |
| --- | --- | --- |
| [`Logging/`](Logging/) | Centralized logging and ELK-based log pipelines | Active |
| [`Vms/`](Vms/) | VM templates, infrastructure as code, and provisioning assets | Planned |
| [`apps/`](apps/) | Application stacks and services | Planned |
| [`containers/`](containers/) | General-purpose container definitions and shared images | Planned |
| [`databases/`](databases/) | Database services and persistent storage definitions | Planned |
| [`homepage/`](homepage/) | Dashboard and service landing page | Active |
| [`monitoring/`](monitoring/) | Metrics, alerts, dashboards, and observability tooling | In progress |
| [`reverse-proxy/`](reverse-proxy/) | Ingress, TLS, and routing configuration | Planned |
| [`security/`](security/) | Vulnerability scanners, telemetry, and security operations | In progress |

## Active services

- [Homepage](homepage/) — service landing page with quick links and integrations.
- [ELK stack](Logging/elk-stack/) — Elasticsearch, Logstash, and Kibana for log collection and analysis.
- [Prometheus](monitoring/prometheus/) — metrics collection and service monitoring.
- [Grafana](monitoring/grafana/) — dashboards and visual observability.
- [Nessus](security/nessus/) — vulnerability scanning workspace.
- [OpenVAS](security/openvas/) — vulnerability assessment environment.
- [Telemetry](security/telemetry/) — security event and system telemetry collection assets.

## Getting started

### Prerequisites

- Docker Engine
- Docker Compose v2 (`docker compose`)
- A Linux host with sufficient CPU, memory, and storage for persistent services
- Optional: virtualization tools for VM workloads in [`Vms/`](Vms/)

### Starting a service

Move into the service directory and follow its local README. A typical workflow for Compose-based services is:

```bash
cd path/to/service
docker compose up -d
docker compose ps
docker compose logs -f
```

To stop a stack:

```bash
docker compose down
```

Do not run Compose from the repository root unless a dedicated root-level stack is added in the future.

## Configuration and secrets

- Keep service-specific configuration near the service that uses it.
- Commit sanitized `.env.sample` files, never populated `.env` files.
- Do not commit passwords, API keys, certificates, private keys, or generated data.
- Back up persistent volumes and document restore steps in the relevant service README.
- Prefer explicit environment variables and versioned configuration files over hidden defaults.

## Contributing

When adding a new service:

1. Create a dedicated directory under the most relevant top-level category.
2. Add a README with purpose, prerequisites, ports, volumes, configuration, and recovery steps.
3. Include a sanitized `.env.sample` if environment variables are required.
4. Document the service and its deployment flow in this README.
5. Keep the stack isolated and avoid coupling unrelated components together.

## License

This repository is released under the [MIT License](LICENSE).

## Roadmap

- Define shared networking and storage conventions across services
- Add reverse-proxy and TLS documentation
- Document backup and restore procedures for persistent stacks
- Add health checks and monitoring targets for deployed services
- Expand the lab with additional app, database, and automation components
