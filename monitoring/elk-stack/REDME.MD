# ELK Stack Lab Environment

This project is a local Docker Compose-based ELK stack designed for log collection, indexing, and visualization in a lab or homelab environment.

## Overview

The stack currently includes:

- Elasticsearch for indexing and searching log data
- Logstash for receiving and forwarding log events
- Kibana for exploring data and building dashboards

The configuration is defined in `docker-compose.yml`, and the Logstash pipeline is configured in `logstash/pipeline/logstash.conf`.

## Current Stack Versions

The compose file is using the following Elastic versions:

- Elasticsearch: `9.5.4`
- Logstash: `9.5.4`
- Kibana: `9.5.4`

## Architecture

```text
Applications / agents
        │
        ▼
   Logstash:5044 / 5000
        │
        ▼
 Elasticsearch:9200
        │
        ▼
   Kibana:5601
```

### Traffic flow

- Logstash listens on `5044` for Beats-style traffic
- Logstash also listens on `5000/tcp` and `5000/udp` for raw JSON input
- Elasticsearch stores all incoming documents
- Kibana connects to Elasticsearch and exposes the UI for search and dashboards

## Services

### Elasticsearch
- Container name: `elasticsearch`
- Image: `docker.elastic.co/elasticsearch/elasticsearch:9.5.4`
- Port: `9200`
- Persistent storage: Docker volume `esdata`
- Network: `elk`

### Logstash
- Container name: `logstash`
- Image: `docker.elastic.co/logstash/logstash:9.5.4`
- Exposed ports:
  - `5044` for Beats input
  - `5000/tcp` for JSON-over-TCP input
  - `5000/udp` for JSON-over-UDP input
- Pipeline directory: `./logstash/pipeline`
- Network: `elk`

### Kibana
- Container name: `kibana`
- Image: `docker.elastic.co/kibana/kibana:9.5.4`
- Port: `5601`
- Connects to Elasticsearch at `http://elasticsearch:9200`
- Network: `elk`

## Project Structure

```text
monitoring/Belk/
├── docker-compose.yml
├── REDME.MD
├── logstash/
│   └── pipeline/
│       └── logstash.conf
└── Docker-managed volumes
```

## Quick Start

From this directory, start the stack:

```bash
docker compose up -d
```

Check the status:

```bash
docker compose ps
```

Watch logs:

```bash
docker compose logs -f
```

Access the interfaces:

- Elasticsearch: `http://localhost:9200`
- Kibana: `http://localhost:5601`

## Logstash Pipeline Behavior

The current pipeline listens for:

- Beats on port `5044`
- JSON payloads over TCP on port `5000`

It forwards each event to Elasticsearch with the index naming pattern:

```text
lab-logs-%{+YYYY.MM.dd}
```

This creates a daily index based on the event timestamp.

The pipeline configuration is:

```conf
input {
  beats {
    port => 5044
  }
  tcp {
    port => 5000
    codec => json
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "lab-logs-%{+YYYY.MM.dd}"
  }
}
```

## Security Notes

This project is currently set up as a single-node Elasticsearch instance using:

```yaml
environment:
  - discovery.type=single-node
```

The security toggle is currently commented out:

```yaml
# - xpack.security.enabled=false
```

That means the default Elasticsearch 9 behavior is secure by default. If you want to use the lab-friendly insecure mode instead, uncomment the line and recreate the volume.

## Setup Options

### Option 1: Complete the secure setup

Use the built-in Elastic enrollment flow if you want to continue with the secure default configuration.

1. Generate the Kibana enrollment token:

```bash
docker exec -it elasticsearch /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

2. Paste the token into the Kibana setup wizard.

3. Retrieve the verification code:

```bash
docker exec -it kibana bin/kibana-verification-code
```

4. Reset the default `elastic` password:

```bash
docker exec -it elasticsearch /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic
```

### Option 2: Force insecure lab mode

If you want the no-auth local lab setup, use the insecure mode and remove the persistent data:

```bash
docker compose down -v
```

Then make sure the compose file includes:

```yaml
environment:
  - discovery.type=single-node
  - xpack.security.enabled=false
```

Finally, bring the stack back up:

```bash
docker compose up -d
```

## Useful Commands

```bash
# Start services
docker compose up -d

# Stop services
docker compose down

# Stop and remove volumes
docker compose down -v

# Show containers
docker compose ps

# View logs
docker compose logs -f

# Check Elasticsearch health
curl http://localhost:9200
```
## Adding log agants

### Winlog
        TODO




## Troubleshooting

If the stack is not healthy:

1. Verify the containers are running with `docker compose ps`
2. Inspect logs with `docker compose logs -f elasticsearch` and `docker compose logs -f kibana`
3. Ensure Elasticsearch responds on `http://localhost:9200`
4. Confirm Logstash is listening on ports `5044` and `5000`
5. If you changed security settings, recreate the persistent data volume as needed

## Notes

- The project is intended for local testing, monitoring labs, and development environments.
- Elasticsearch data is stored in a Docker volume to survive container restarts.
- The current configuration is a single-node setup, so it is best suited for local or lab workloads rather than production deployments.

## License

This project is intended for local monitoring and lab use.


