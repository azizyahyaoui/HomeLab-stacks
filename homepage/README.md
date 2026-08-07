# Homepage

## Overview

`Homepage` is a lightweight homelab dashboard frontend designed to give you a single-pane view of your local infrastructure. The dashboard consolidates servers, Docker management, monitoring integrations, bookmarks, widgets and quick links into a clean, customizable UI.

The project in this repository provides the frontend and deployment helpers used to run the dashboard locally or in containers.

## What is included

- Servers panel (Proxmox, local host summaries)
- Docker Management cards (Portainer, agent status)
- Monitoring integration (e.g. Wazuh)
- Quick access sections (Work, Sites, Entertainment)
- Configuration directory (`config/`) for runtime settings
- Bookmarks and widgets definitions (`bookmarks.yaml`, `widgets.yaml`)
- Customization files (`custom.css`, `custom.js`) and background images (`assets/images/background/`)
- Deployment manifests: `compose.yml`, `docker.yaml`, `kubernetes.yaml`, `proxmox.yaml`, `services.yaml`

You can see a sample screenshot in the repository assets (or your running instance). Example screenshot attached in the project root.

## Quick start (Docker Compose)

1. Copy the environment template and edit it locally:

```bash
cp .env.sample .env
# Edit .env to set your local values (port, PUID/PGID, timezone, etc.)
```

2. Start the service:

```bash
docker compose up -d
```

3. View logs:

```bash
docker compose logs -f
```

4. Stop the service:

```bash
docker compose down
```

## Important: Use `.env.sample` as the template

- This repository purposely references `.env.sample` as the environment template. Do not commit a populated `.env` file into version control — keep secrets out of the repo.
- Workflow: edit `.env.sample` locally, then create a private `.env` by copying the sample (`cp .env.sample .env`) and filling in real values.
- If a `.env.sample` file is not present, create one from your local `.env` by removing secrets and committing the sanitized file.

Example variables you may find in `.env.sample`:

```env
# Example .env.sample
HOMEPAGE_PORT=3000
TZ=UTC
PUID=1000
PGID=1000
# Add any integration keys here as placeholders (do NOT commit real keys)
```

## Configuration and data directories

- `config/` — persistent application configuration. Keep backups if you make manual edits.
- `assets/` — images and static media used by the frontend.
- `bookmarks.yaml` — shortcuts and quick links displayed in the UI.
- `widgets.yaml` and `settings.yaml` — widget definitions and UI settings.

## Deploying with Kubernetes or other platforms

- Use `kubernetes.yaml` for a basic Kubernetes deployment example.
- `docker.yaml` and `compose.yml` contain Docker-based deployment options.
- Adjust volume and secret mounts to provide runtime configuration without committing secrets to the repository.

## Security & Best practices

- Never commit `.env` to the repository. Commit only `.env.sample`.
- Limit access to any secrets and integration credentials.
- Mount `/var/run/docker.sock` read-only where possible and only if you trust the service and host.

## Troubleshooting

- If the dashboard is unreachable, check the port in your `.env` file and ensure the container is running (`docker compose ps`).
- Use `docker compose logs -f` to view runtime errors.

## Contributing

- Open issues or PRs for bug fixes and feature requests.
- Keep environment secrets out of PRs — use `.env.sample` for examples.

## License

See the project license (if any) or add one before publishing.

---

File: [README.md](README.md)

