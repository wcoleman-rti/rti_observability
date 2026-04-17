# Getting Started with RTI Observability

## 1. Dependencies

- RTI Connext 7.3
- Docker
- Docker images (pulled automatically if using rtiobservability or docker compose)
  - [rticom/collector-service](https://hub.docker.com/r/rticom/collector-service)
  - [prom/prometheus](https://hub.docker.com/r/prom/prometheus)
  - [grafana/grafana-enterprise](https://hub.docker.com/r/grafana/grafana-enterprise)
  - [grafana/loki](https://hub.docker.com/r/grafana/loki)

## 2. Start App(s) with Monitoring (2.0)

```bash
NDDS_QOS_PROFILES=config/FactoryQos.xml <app>
```

For example, you can run RTI DDS Ping as a test:

```bash
NDDS_QOS_PROFILES=config/FactoryQos.xml $NDDSHOME/bin/rtiddsping -pub
```

## 3. Launch

There are 4 Docker containers/services that need to be running:

1. RTI Collector Service Docker container (for collecting RTI metrics and pushing to telemetry [Prometheus/Loki] backends)
2. Loki Docker container (for storing logs)
3. Prometheus Docker container (for storing metrics)
4. Grafana Docker container (for visualizing logs & metrics)

You can launch them 1 of 3 ways:

1. With rtiobservability tool (best if you want to launch new Grafana/Prometheus/Loki services locally)
2. With docker compose (best if you want to see what the rtiobservability tool does at the docker compose level)
3. With docker (best if you want to use docker directly alongside existing Grafana/Prometheus/Loki services, i.e. just launch RTI Collector Service)

### 1. Run with rtiobservability

Use rtiobservability tool to set up the infrastructure based on your configuration:

```bash
$NDDSHOME/bin/rtiobservability -c config/rti_observability.json
```

The following command starts all 4 docker containers and creates the Docker data volumes for data storage for:

1. Prometheus
2. Grafana

```bash
$NDDSHOME/bin/rtiobservability -i
```

To just start the containers (and avoid re-creating the data volumes if already ran previously with -i):

```bash
$NDDSHOME/bin/rtiobservability -s
```

### 2. Run with Docker Compose

You still need to scaffold the docker-compose.yml, so we will use rtiobservability tool to set up the infrastructure:

```bash
$NDDSHOME/bin/rtiobservability -c config/rti_observability.json
```

Start the containers:

```bash
docker compose -f ~/rti_workspace/7.3.0/user_config/observability/docker-compose.yml up -d
```

### 3. Run with Docker

```bash
# Start RTI Collector Service
#   - OBSERVABILITY_DOMAIN: The domain ID in which Collector Service receives the telemetry data from Connext applications.
#   - OBSERVABILITY_PROMETHEUS_EXPORTER_PORT: The TCP port of the Prometheus endpoint for exporting telemetry data to Prometheus.
#   - OBSERVABILITY_LOKI_HOSTNAME: The hostname where the Loki server runs.
#   - OBSERVABILITY_LOKI_EXPORTER_PORT: The TCP port of the Grafana Loki server in which it will receive logs from Collector Service.
# See: https://github.com/rticommunity/rticonnextdds-containers/blob/release/7.3.0/doc/collector-service/README.md#built-in-configuration
docker run -dt \
        --network host \
        -v $RTI_LICENSE_FILE:/opt/rti.com/rti_connext_dds-7.3.0/rti_license.dat \
        --name=collector_service \
        -e CFG_NAME="NonSecureLAN" \
        -e OBSERVABILITY_DOMAIN=120 \
        -e OBSERVABILITY_PROMETHEUS_EXPORTER_PORT=19090 \
        -e OBSERVABILITY_LOKI_HOSTNAME=localhost \
        -e OBSERVABILITY_LOKI_EXPORTER_PORT=3100 \
        rticom/collector-service:7.3.0
```

## 4. Verify Container Statuses

Check the status of the launched containers for startup failures:

```bash
docker ps -a
```

*Refer to [Docker Container[s] Failed to Start](https://community.rti.com/static/documentation/connext-dds/7.3.0/doc/manuals/addon_products/observability/troubleshooting.html#docker-container-s-failed-to-start) if any fail to start.*

## 5. Visualize Data in Grafana

Open: [http://localhost:3000](http://localhost:3000)

- Username: admin
- Password: admin

*Refer to [Configure Grafana](https://community.rti.com/static/documentation/connext-dds/7.3.0/doc/manuals/addon_products/observability/install.html#configure-grafana).*

## 6. Stop

Depending on how you launched them, you should use the same method to stop/remove running containers.

### 1. Stop with rtiobservability

Just stop:

```bash
$NDDSHOME/bin/rtiobservability -t
```

Stop and Remove:

```bash
$NDDSHOME/bin/rtiobservability -d
```

### 2. Stop with Docker Compose

```bash
docker compose -f ~/rti_workspace/7.3.0/user_config/observability/docker-compose.yml stop
```

```bash
docker compose -f ~/rti_workspace/7.3.0/user_config/observability/docker-compose.yml down
```

### 3. Stop with Docker

```bash
docker stop collector_service
```

```bash
docker rm collector_service
```
