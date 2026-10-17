# Local ClearML Server

Docker Compose deployment for a local ClearML management server, adapted from the [official ClearML Server v2.4.0 Compose file](https://github.com/clearml/clearml-server/blob/v2.4.0/docker/docker-compose.yml).

This project runs the API server, web UI, file server, MongoDB, Redis, Elasticsearch, and the asynchronous file-deletion service. Training agents run separately. Service images use fixed version tags, and persistent data is stored in directories relative to `compose.yaml`.

## Requirements

- Linux with Docker Engine and the Docker Compose plugin.
- Enough memory for all services. The current Elasticsearch settings were chosen for a 64 GB machine shared with training workloads: a 4 GB JVM heap and an 8 GB container memory limit. These are deployment settings, not a minimum-memory specification.
- Available local ports `8008`, `8080`, and `8081`.

## Initial setup

From the cloned project directory, create the directories used by bind mounts:

```bash
mkdir -p config logs data/fileserver data/elastic_7 \
  data/mongo_4/db data/mongo_4/configdb data/redis
```

Configure the kernel setting needed by Elasticsearch and set ownership of its data directory:

```bash
echo 'vm.max_map_count=524288' | sudo tee /etc/sysctl.d/99-clearml.conf
sudo sysctl -w vm.max_map_count=524288
sudo chown -R 1000:1000 data/elastic_7
```

This Compose file uses literal configuration values and does not require a `.env` file. The `config/` directory can hold local ClearML Server configuration overrides and is excluded from Git.

## Start and stop

```bash
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail 100 -f
```

The first startup may take several minutes to download images and initialize databases. A running container does not by itself confirm that its service is ready. Omit `sudo` if your user has Docker access.

| Service | Local address |
| --- | --- |
| Web UI | http://localhost:8080 |
| API | http://localhost:8008 |
| File server | http://localhost:8081 |

Create API credentials in the web UI under **Settings > Workspace** and configure them in your training clients and agents. Do not commit credentials to this repository.

Stop the server with:

```bash
sudo docker compose down --timeout 120
```

Persistent data remains in the local `data/` directory.

## Local access and remote workers

Published ports bind to `127.0.0.1`, so this configuration accepts clients on the same machine only. It does not enable access from other training servers.

For remote access, configure a stable server address, authentication, firewall rules, and TLS as needed. Update port bindings and the file-server URL settings in Compose consistently. Clients and agents must use addresses reachable from their machines; `localhost` on a remote training host points to that host, not this management server. Do not expose the default deployment to an untrusted network without access controls.

## Persistent files and migration

| Path | Contents | Included in Git |
| --- | --- | --- |
| `compose.yaml` | Service definitions and image versions | Yes |
| `data/` | Databases, experiment events, and uploaded files | No |
| `config/` | Local server configuration overrides | No |
| `logs/` | Server logs | No |
| `.env` and local environment variants | Optional local settings and credentials | No |
| `compose.yaml.before-*` | Local historical Compose backups | No |

A Git clone contains deployment files, not existing experiments or server state. For migration, stop training clients and the server first, then copy the deployment directory including ignored files, hidden files, permissions, and ownership. Start with the same image versions and CPU architecture on the destination. Configure Docker and the kernel setting there as well. Docker images are not stored in this directory and must be downloaded or transferred separately. Models and datasets stored outside this directory need their own backup.

## Elasticsearch memory

The Compose file sets:

```yaml
ES_JAVA_OPTS: "-Xms4g -Xmx4g"
mem_limit: 8g
```

The container limit includes memory beyond the JVM heap. After editing these values, recreate only Elasticsearch:

```bash
sudo docker compose up -d --no-deps elasticsearch
```

Metrics recording and queries may be delayed while Elasticsearch restarts. Other services and training containers are not restarted by this command.

Check cluster health and resource usage:

```bash
sudo docker compose exec -T elasticsearch curl -fsS \
  'http://localhost:9200/_cluster/health?wait_for_status=yellow&timeout=60s'
sudo docker stats --no-stream
```

Revisit memory settings as experiment volume and workloads change.
