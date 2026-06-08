# Getting Started with RTI Observability

## 1. Dependencies

- RTI Connext 7.3
- Docker
- Docker images (pulled automatically if using rtiobservability or docker compose)
  - [rticom/collector-service](https://hub.docker.com/r/rticom/collector-service)
  - [prom/prometheus](https://hub.docker.com/r/prom/prometheus)
  - [grafana/grafana-enterprise](https://hub.docker.com/r/grafana/grafana-enterprise)

## 2. Start App(s) with Monitoring 2.0

```bash
NDDS_QOS_PROFILES=qos/FactoryQos.xml <app>
```

For example, you can run RTI DDS Ping as a test:

```bash
NDDS_QOS_PROFILES=qos/FactoryQos.xml $NDDSHOME/bin/rtiddsping -pub
```

## 3. Launch

There are 3 Docker containers/services that need to be running:

1. RTI Collector Service Docker container (for collecting RTI metrics and pushing to telemetry [Prometheus] backends)
2. Prometheus Docker container (for storing metrics)
3. Grafana Docker container (for visualizing logs & metrics)

In this guide, you can launch them 1 of 2 ways:

1. With `docker run`
2. With `docker compose`

### 1. Run with `docker run`

Run RTI Collector Service:

```bash
# Start RTI Collector Service
#   - OBSERVABILITY_DOMAIN: The domain ID in which Collector Service receives the telemetry data from Connext applications.
#   - OBSERVABILITY_PROMETHEUS_EXPORTER_PORT: The TCP port of the Prometheus endpoint for exporting telemetry data to Prometheus.
#   - CFG_NAME: see https://github.com/rticommunity/rticonnextdds-containers/blob/release/7.3.1/doc/collector-service/README.md#built-in-configuration
docker run -dt \
        --network host \
        --ipc host \
        -v $RTI_LICENSE_FILE:/opt/rti.com/rti_connext_dds-7.3.1/rti_license.dat \
        --name collector_service \
        -e CFG_NAME="NonSecureLAN" \
        -e OBSERVABILITY_DOMAIN=101 \
        -e OBSERVABILITY_PROMETHEUS_EXPORTER_PORT=19090 \
        rticom/collector-service:7.3.1.3
```

Run Prometheus:

```bash
# Start Prometheus
docker run -dt \
        --network host \
        -v ./prometheus:/etc/prometheus \
        -v prometheus_data:/prometheus \
        --name=prometheus_observability \
        prom/prometheus:v2.45.1 \
        --config.file=/etc/prometheus/prometheus.yml \
        --storage.tsdb.path=/prometheus \
        --web.console.libraries=/etc/prometheus/console_libraries \
        --web.console.templates=/etc/prometheus/consoles \
        --storage.tsdb.retention.time=200h \
        --query.lookback-delta=30s \
        --web.enable-lifecycle \
        --web.enable-admin-api \
        --web.listen-address=0.0.0.0:9090
```

Run Grafana:

```bash
# Start Grafana
docker run -dt \
        --network host \
        -v grafana_data:/var/lib/grafana \
        -v ./grafana/dashboards:/etc/grafana/provisioning/dashboards \
        -v ./grafana/datasources:/etc/grafana/provisioning/datasources \
        -v ./img/RTI_Logo_RGB-Color.png:/usr/share/grafana/public/img/RTI_Logo_RGB-Color.png \
        --name=grafana_observability \
        -e GF_SECURITY_ADMIN_USER=${ADMIN_USER:-admin} \
        -e GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin} \
        -e GF_USERS_ALLOW_SIGN_UP=false \
        -e GF_SERVER_HTTP_PORT=3000 \
        -e GF_SERVER_DOMAIN=localhost \
        -e GF_USERS_HOME_PAGE="/d/Hj4nnyOnk/alert-home?orgId=1&refresh=10s" \
        -e GF_INSTALL_PLUGINS="https://github.com/VolkovLabs/volkovlabs-form-panel/releases/download/v3.2.1/volkovlabs-form-panel-3.2.1.zip;volkovlabs-form-panel" \
        grafana/grafana-enterprise:10.1.4
```

### 2. Run with `docker compose`

Start the containers:

```bash
docker compose up -d
```

## 4. Verify Container Statuses

Check the status of the launched containers for startup failures:

```bash
docker ps -a
```

*Refer to [Docker Container[s] Failed to Start](https://community.rti.com/static/documentation/connext-dds/7.3.1/doc/manuals/addon_products/observability/troubleshooting.html#docker-container-s-failed-to-start) if any fail to start.*

## 5. Visualize Data in Grafana

Open: [http://localhost:3000](http://localhost:3000)

- Username: admin
- Password: admin

*Refer to [Configure Grafana](https://community.rti.com/static/documentation/connext-dds/7.3.1/doc/manuals/addon_products/observability/install.html#configure-grafana).*

## 6. Stop

Depending on how you launched them, you should use the same method to stop/remove running containers.

### 1. Stop with `docker stop`

To stop:

```bash
# To stop:
docker stop collector_service prometheus_observability grafana_observability

# To remove:
docker rm collector_service prometheus_observability grafana_observability
```

### 2. Stop with `docker compose`

```bash
# To stop:
docker compose stop

# To remove:
docker compose down
```
