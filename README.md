# Getting Started with RTI Observability

## 1. Dependencies

- RTI Connext 7.3 (monitored applications)
- Docker
- Docker images (pulled automatically if using rtiobservability or docker compose)
  - [rticom/collector-service](https://hub.docker.com/r/rticom/collector-service) (>= 7.7.0 required)
  - [prom/prometheus](https://hub.docker.com/r/prom/prometheus)
  - [grafana/grafana-enterprise](https://hub.docker.com/r/grafana/grafana-enterprise)
  - [rticom/dds-ping](https://hub.docker.com/r/rticom/dds-ping) *(optional — built-in pub/sub pair for testing)*

## 2. Start App(s) with Monitoring 2.0

Launch your application with all three QoS files on `NDDS_QOS_PROFILES` and set
`DDS_MONITOR_PROFILE` to select a monitoring tier (default: `Production`):

```bash
DDS_MONITOR_PROFILE=Production \
  NDDS_QOS_PROFILES="qos/Monitoring.xml;qos/Logging.xml;qos/FactoryQos.xml" \
  <app>
```

For example, run RTI DDS Ping as a test:

```bash
DDS_MONITOR_PROFILE=Production \
  NDDS_QOS_PROFILES="qos/Monitoring.xml;qos/Logging.xml;qos/FactoryQos.xml" \
  $NDDSHOME/bin/rtiddsping -pub
```

Running a subscriber alongside it on the **same host** works with the same command:

```bash
DDS_MONITOR_PROFILE=Production \
  NDDS_QOS_PROFILES="qos/Monitoring.xml;qos/Logging.xml;qos/FactoryQos.xml" \
  $NDDSHOME/bin/rtiddsping -sub
```

**Alternatively**, use the built-in `dds-ping` Docker Compose profile to start a
publisher and subscriber pair automatically alongside the observability stack:

```bash
# Start everything (observability stack + dds-ping pub/sub)
docker compose --profile dds-ping up -d

# Choose a monitoring tier (default: Production)
DDS_MONITOR_PROFILE=Development docker compose --profile dds-ping up -d
```

See step 3 for details on the `dds-ping` profile.

### Monitoring Tiers

Three profiles are available. Set `DDS_MONITOR_PROFILE` to the profile name.

| Profile | Polling | Event pub | Log level | Use when |
|---|---|---|---|---|
| `Production` | 15 s | 1 s | WARNING | Steady-state, low overhead |
| `Development` | 5 s | immediate | INFORMATIONAL | Active debugging, QoS tuning |
| `Discovery` | 1 s | immediate | INFORMATIONAL | Startup timing, match/unmatch analysis |

**Production** — collects reliable-cache saturation, event counters (liveliness,
deadline, QoS incompatibility, publication/subscription matched), DataReader
protocol traffic, cache state, memory, and UDP network metrics. WARNING logs only.

**Development** — same as Production, plus DataReader sample-rejection counts and
INFORMATIONAL logs on all channels.

**Discovery** — lightweight: only QoS incompatibility and publication/subscription
matched events (writers and readers), plus baseline memory and network. INFORMATIONAL
middleware logs. Use only during startup or convergence analysis — 1 s polling
generates ~15× more Prometheus samples than Production.

> **Grafana** — each tier has a dedicated dashboard. See step 5.

### Monitoring Participant Discovery

The dedicated Monitoring Library 2.0 participant (domain 101) uses multicast
discovery (`239.255.0.1`) and RTI's built-in fast-discovery QoS snippets for
quick startup, and sets a `domain_tag` required for compatibility with RTI
Collector Service >= 7.7.0. If Collector Service is on a different
machine/subnet where this multicast group isn't routed, or you're using
Collector Service < 7.7.0, see the `MonitoringParticipant` profile comments
in `qos/Monitoring.xml` for how to adjust.

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
        -v $RTI_LICENSE_FILE:/opt/rti.com/rti_license.dat \
        --name collector_service \
        -e CFG_NAME="NonSecureLAN" \
        -e OBSERVABILITY_DOMAIN=101 \
        -e OBSERVABILITY_PROMETHEUS_EXPORTER_PORT=19090 \
        -e RTI_LICENSE_FILE=/opt/rti.com/rti_license.dat \
        rticom/collector-service:7.7.0
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

#### Observability stack only

```bash
docker compose up -d
```

#### Observability stack + dds-ping test pair

The `dds-ping-pub` and `dds-ping-sub` services are gated behind the `dds-ping`
Compose profile so they do not start by default.

```bash
# Start everything
docker compose --profile dds-ping up -d

# Select a monitoring tier (env var is forwarded into both dds-ping containers)
DDS_MONITOR_PROFILE=Development docker compose --profile dds-ping up -d
```

The dds-ping containers:
- Run on **DDS domain 0** (data plane)
- Load QoS from `./qos/` (Monitoring + Logging + Factory)
- Forward Monitoring Library 2.0 telemetry to Collector Service on **domain 101**
- Default topic: `RTIddsPingTopic` / type: `RTIddsPingType`, best-effort

> **Tip:** with both pub and sub running, the match-status panels
> (`publication_matched_current_count`, `subscription_matched_current_count`)
> in the tier dashboards will populate and show matched endpoint counts.

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

### Tier Dashboards

Three dashboards are provisioned automatically and placed in named Grafana folders
matching the monitoring tier. Navigate to the folder for the profile your application
is running.

| Grafana folder | Dashboard | Profile | Panels |
|---|---|---|---|
| **Production** | RTI Connext — Production Tier | `Production` | Health rollup, data loss, delays, writer cache saturation, match/liveliness, DataReader traffic, reader cache, resources |
| **Development** | RTI Connext — Development Tier | `Development` | All Production panels + sample rejection |
| **Discovery** | RTI Connext — Discovery Tier | `Discovery` | QoS compatibility, publication/subscription matched events, resources |

The existing general-purpose dashboards (participant/writer/reader detail, alert
summaries, log control, etc.) remain in the root **General** folder.

> **Note:** If the tier dashboards show no data, verify that `DDS_MONITOR_PROFILE`
> is set to the matching profile name for the running application(s).

## 6. Stop

Depending on how you launched them, you should use the same method to stop/remove running containers.

### 1. Stop with `docker stop`

```bash
# To stop:
docker stop collector_service prometheus_observability grafana_observability \
       dds_ping_pub dds_ping_sub

# To remove:
docker rm collector_service prometheus_observability grafana_observability \
      dds_ping_pub dds_ping_sub
```

### 2. Stop with `docker compose`

```bash
# Stop observability stack only:
docker compose stop

# Stop everything including dds-ping:
docker compose --profile dds-ping stop

# Remove all containers:
docker compose --profile dds-ping down
```
