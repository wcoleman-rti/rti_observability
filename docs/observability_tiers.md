# RTI Connext Professional 7.3.1 — Tiered Observability

This document describes the three-tier Monitoring Library 2.0 observability
strategy for this project.

---

## Overview

Three monitoring tiers are implemented via `participant_factory_qos` profiles in
`qos/Monitoring.xml`, controlled by the `DDS_MONITOR_PROFILE` environment variable:

| Tier | Polling | Event pub | Log level | Overhead | Use case |
|---|---|---|---|---|---|
| `Production` | 15 s | 1 s | WARNING | Low | Steady-state production |
| `Development` | 5 s | immediate | INFORMATIONAL | Medium | Active debugging, QoS tuning |
| `Discovery` | 1 s | immediate | INFORMATIONAL | High | Startup, convergence analysis |

All three:
- Enable Monitoring Library 2.0 via `BuiltinQosSnippetLib::Feature.Monitoring2.Enable`
- Send telemetry to RTI Collector Service on domain 101 (configured as `OBSERVABILITY_DOMAIN=101` in `docker-compose.yml`)
- Collector Service exports to Prometheus on port 19090
- Use multicast discovery (239.255.0.1) with fast-discovery QoS optimizations
- Configure the dedicated monitoring participant with domain tag `RTI_o11y` (required for Collector Service >= 7.7.0)

---

## How to Use Each Tier

### 1. Production (Default)

Activate via environment variable (most common approach for containerized apps):

```bash
DDS_MONITOR_PROFILE=Production \
  NDDS_QOS_PROFILES="qos/Monitoring.xml;qos/Logging.xml;qos/FactoryQos.xml" \
  <app>
```

Or with docker-compose (env var forwarded to dds-ping containers):

```bash
DDS_MONITOR_PROFILE=Production docker compose --profile dds-ping up
```

**Grafana dashboard**: [http://localhost:3000](http://localhost:3000) → Production folder

### 2. Development

For active debugging, QoS tuning, sample-rejection analysis:

```bash
DDS_MONITOR_PROFILE=Development \
  NDDS_QOS_PROFILES="qos/Monitoring.xml;qos/Logging.xml;qos/FactoryQos.xml" \
  <app>
```

**Grafana dashboard**: Development folder

### 3. Discovery

For startup timing, match/unmatch churn analysis. Use only during investigation — 1 s polling
generates ~15× more Prometheus samples than Production.

```bash
DDS_MONITOR_PROFILE=Discovery \
  NDDS_QOS_PROFILES="qos/Monitoring.xml;qos/Logging.xml;qos/FactoryQos.xml" \
  <app>
```

**Grafana dashboard**: Discovery folder (includes a warning to switch tiers after analysis)

---

## Configuration Files

### qos/Monitoring.xml

Contains five QoS profiles:

1. **`Main`** — Default participant factory QoS
   - Selects a snippet profile via `$(DDS_MONITOR_PROFILE)` env var
   - Configures the Monitoring Library 2.0 dedicated participant (domain 101)
   - Sets `collector_initial_peers` to `239.255.0.1` (multicast address)

2. **`Production`**, **`Development`**, **`Discovery`** — Snippet profiles
   - Define polling intervals, event batching, log levels
   - Included via `<?rti-qos_snippet?>`

3. **`MonitoringParticipant`** — Dedicated participant profile
   - Extends `BuiltinQosLib::Generic.Monitoring2`
   - Adds fast-discovery QoS snippets
   - Enables multicast discovery on 239.255.0.1
   - Sets `dds.domain_participant.domain_tag` = `RTI_o11y` (Collector Service 7.7.0 requirement)

---

## Monitoring Participant Discovery

The dedicated Monitoring Library 2.0 participant:
- Runs on **domain 101** (separate from application domain)
- Uses **multicast 239.255.0.1** for discovery
- Includes RTI fast-discovery QoS snippets for rapid convergence
- Sets **domain_tag = `RTI_o11y`** (required for Collector Service >= 7.7.0)

**Multicast troubleshooting**: If Collector Service is on a different subnet/VLAN
where multicast 239.255.0.1 isn't routed, replace the `collector_initial_peers`
entry in `qos/Monitoring.xml`'s `Main` profile with an explicit unicast address
(e.g., `<element>192.168.1.2</element>`).

**Collector Service < 7.7.0**: Comment out the `<property>` element in the
`MonitoringParticipant` profile that sets `domain_tag`.

---

## Application Naming

Monitoring Library 2.0 auto-generates a unique application resource identifier per process:

```
/applications/<host_name:process_id:uuid>
```

This ensures no collisions even when multiple monitored processes run on the same host.
The `application_name` field is left empty intentionally; explicitly setting it is optional
and purely for human-readable naming in Grafana.

---

## Dashboards

Three tier dashboards are provisioned automatically:

| Tier | File | Location | Panels |
|---|---|---|---|
| Production | `Production/production_dashboard.json` | Grafana → Production folder | Health rollup, data loss, delays, cache saturation, match/liveliness |
| Development | `Development/development_dashboard.json` | Grafana → Development folder | All Production panels + sample rejection counts |
| Discovery | `Discovery/discovery_dashboard.json` | Grafana → Discovery folder | QoS incompatibility, publication/subscription matched events, warnings |

General-purpose alert and status dashboards remain in the **General** folder.

---

## Docker Deployment

`docker-compose.yml` orchestrates:
- **Collector Service 7.7.0** (required for this configuration)
- **Prometheus** (time-series storage)
- **Grafana** (visualization)
- **dds-ping** services (optional test pub/sub pair, gated behind `dds-ping` profile)

```bash
# Start the full observability stack
docker compose up -d

# Start with dds-ping test pair
docker compose --profile dds-ping up -d
```

---

## Troubleshooting

**Applications not appearing in Collector Service /metrics**:
1. Verify `NDDS_QOS_PROFILES` includes `qos/Monitoring.xml` and loads before participant creation
2. Check that Collector Service is reachable on multicast 239.255.0.1 or the unicast address in `collector_initial_peers`
3. Confirm `OBSERVABILITY_DOMAIN=101` in Collector Service matches domain 101 in `qos/Monitoring.xml`

**Missing metrics in Grafana**:
1. Verify `DDS_MONITOR_PROFILE` matches the tier name (case-sensitive)
2. Check Prometheus scrape targets at [http://localhost:9090/targets](http://localhost:9090/targets)
3. Wait for initial collection interval (15 s for Production, 5 s for Development, 1 s for Discovery)

**High resource usage**:
1. If Discovery tier is active, switch to Production or Development after analysis
2. Discovery polling (1 s) generates ~15× more Prometheus samples; sustained use impacts storage and CPU

### Verified Prometheus Metric Names

All names below are confirmed from `prometheus/monitoring_recording_rules.yml`
and `grafana/dashboards/General/*.json` in this project.

#### DataWriter (protocol)
| Prometheus metric | Description |
|---|---|
| `dds_data_writer_protocol_pushed_samples_total` | Cumulative pushed samples |
| `dds_data_writer_protocol_pushed_sample_bytes_total` | Cumulative pushed bytes |
| `dds_data_writer_protocol_pulled_samples_total` | Cumulative pulled samples (durable/transient) |
| `dds_data_writer_protocol_pulled_sample_bytes_total` | Cumulative pulled bytes |
| `dds_data_writer_protocol_pushed_fragments_total` | Cumulative pushed fragments |
| `dds_data_writer_protocol_pushed_fragment_bytes_total` | Cumulative pushed fragment bytes |
| `dds_data_writer_protocol_pulled_fragments_total` | Cumulative pulled fragments |
| `dds_data_writer_protocol_pulled_fragment_bytes_total` | Cumulative pulled fragment bytes |
| `dds_data_writer_protocol_sent_heartbeats_total` | Cumulative heartbeats sent |
| `dds_data_writer_protocol_received_nacks_total` | Cumulative NACKs received |
| `dds_data_writer_protocol_received_nack_bytes_total` | Cumulative NACK bytes received |
| `dds_data_writer_protocol_received_nack_fragments_total` | Cumulative NACK fragments received |
| `dds_data_writer_protocol_received_nack_fragment_bytes_total` | Cumulative NACK fragment bytes |
| `dds_data_writer_protocol_send_window_size` | Current send window size (gauge) |

#### DataWriter (cache / reliability)
| Prometheus metric | Description |
|---|---|
| `dds_data_writer_cache_samples` | Current samples in writer cache |
| `dds_data_writer_cache_samples_peak` | Peak samples in writer cache |
| `dds_data_writer_cache_alive_instances` | Current alive instances |
| `dds_data_writer_cache_alive_instances_peak` | Peak alive instances |
| `dds_data_writer_reliable_cache_unack_samples` | Current unacknowledged samples |
| `dds_data_writer_reliable_cache_unack_samples_peak` | Peak unacknowledged samples |
| `dds_data_writer_reliable_cache_full_total` | Cache-full events |
| `dds_data_writer_reliable_cache_high_watermark_total` | High watermark events |
| `dds_data_writer_reliable_cache_replaced_unack_samples_total` | Unack samples replaced (data loss indicator) |
| `dds_data_writer_reliable_reader_activity_inactive_count` | Reliable readers inactive (churn) |

#### DataWriter (status)
| Prometheus metric | Description |
|---|---|
| `dds_data_writer_liveliness_lost_total` | Liveliness lost events |
| `dds_data_writer_deadline_missed_total` | Deadline missed events |
| `dds_data_writer_incompatible_qos_total` | QoS incompatibility events |
| `dds_data_writer_presence` | Entity presence (0/1) |

#### DataReader (protocol)
| Prometheus metric | Description |
|---|---|
| `dds_data_reader_protocol_received_samples_total` | Cumulative received samples |
| `dds_data_reader_protocol_received_sample_bytes_total` | Cumulative received bytes |
| `dds_data_reader_protocol_received_fragments_total` | Cumulative received fragments |
| `dds_data_reader_protocol_reassembled_samples_total` | Cumulative reassembled samples |
| `dds_data_reader_protocol_received_heartbeats_total` | Cumulative heartbeats received |
| `dds_data_reader_protocol_sent_nacks_total` | Cumulative NACKs sent |
| `dds_data_reader_protocol_sent_nack_bytes_total` | Cumulative NACK bytes sent |
| `dds_data_reader_protocol_sent_nack_fragments_total` | Cumulative NACK fragments sent |
| `dds_data_reader_protocol_sent_nack_fragment_bytes_total` | Cumulative NACK fragment bytes |
| `dds_data_reader_protocol_duplicate_samples_total` | Cumulative duplicate samples |
| `dds_data_reader_protocol_duplicate_sample_bytes_total` | Cumulative duplicate bytes |
| `dds_data_reader_protocol_rejected_samples_total` | Cumulative rejected samples |
| `dds_data_reader_protocol_out_of_range_rejected_samples_total` | Out-of-range rejected |
| `dds_data_reader_protocol_dropped_fragments_total` | Dropped fragments |

#### DataReader (cache)
| Prometheus metric | Description |
|---|---|
| `dds_data_reader_cache_samples` | Current samples in reader cache |
| `dds_data_reader_cache_samples_peak` | Peak samples in reader cache |
| `dds_data_reader_cache_alive_instances` | Alive instances |
| `dds_data_reader_cache_alive_instances_peak` | Peak alive instances |
| `dds_data_reader_cache_no_writers_instances` | No-writers instances |
| `dds_data_reader_cache_no_writers_instances_peak` | Peak no-writers instances |
| `dds_data_reader_cache_disposed_instances` | Disposed instances |
| `dds_data_reader_cache_disposed_instances_peak` | Peak disposed instances |
| `dds_data_reader_cache_replaced_dropped_samples_total` | Samples dropped due to replacement |
| `dds_data_reader_cache_samples_dropped_by_instance_replaced_total` | Instance-replace drops |
| `dds_data_reader_cache_tolerance_source_ts_dropped_samples_total` | Timestamp-tolerance drops |
| `dds_data_reader_cache_old_source_ts_dropped_samples_total` | Old-timestamp drops |
| `dds_data_reader_cache_content_filter_dropped_samples_total` | Content-filter drops |
| `dds_data_reader_cache_compressed_samples_total` | Compressed samples |

#### DataReader (status)
| Prometheus metric | Description |
|---|---|
| `dds_data_reader_sample_lost_total` | Samples lost |
| `dds_data_reader_sample_rejected_total` | Samples rejected |
| `dds_data_reader_deadline_missed_total` | Deadline missed events |
| `dds_data_reader_incompatible_qos_total` | QoS incompatibility events |
| `dds_data_reader_liveliness_not_alive_count` | Not-alive liveliness count |
| `dds_data_reader_does_not_exist_total` | Does-not-exist events (discovery timing) |
| `dds_data_reader_presence` | Entity presence (0/1) |

#### Application
| Prometheus metric | Description |
|---|---|
| `dds_application_presence` | Application presence (0/1) |
| `dds_application_process_memory_usage_resident_memory_bytes` | RSS in bytes |
| `dds_application_process_memory_usage_virtual_memory_bytes` | VSZ in bytes |
| `dds_application_logging_forwarding_middleware_level` | Middleware log forwarding level |
| `dds_application_logging_forwarding_security_event_level` | Security log forwarding level |
| `dds_application_logging_forwarding_service_level` | Service log forwarding level |
| `dds_application_logging_forwarding_user_level` | User log forwarding level |
| `dds_application_logging_collection_middleware_level` | Middleware collection level |
| `dds_application_logging_collection_user_level` | User collection level |

#### DomainParticipant
| Prometheus metric | Description |
|---|---|
| `dds_domain_participant_presence` | Participant presence (0/1) |
| `dds_domain_participant_udpv4_usage_in_net_pkts_mean` | UDP4 inbound packets (mean) |
| `dds_domain_participant_udpv4_usage_in_net_bytes_mean` | UDP4 inbound bytes (mean) |
| `dds_domain_participant_udpv4_usage_out_net_pkts_mean` | UDP4 outbound packets (mean) |
| `dds_domain_participant_udpv4_usage_out_net_bytes_mean` | UDP4 outbound bytes (mean) |
| `dds_domain_participant_udpv6_usage_in_net_pkts_mean` | UDP6 inbound packets (mean) |
| `dds_domain_participant_udpv6_usage_in_net_bytes_mean` | UDP6 inbound bytes (mean) |
| `dds_domain_participant_udpv6_usage_out_net_pkts_mean` | UDP6 outbound packets (mean) |
| `dds_domain_participant_udpv6_usage_out_net_bytes_mean` | UDP6 outbound bytes (mean) |

#### Topic
| Prometheus metric | Description |
|---|---|
| `dds_topic_presence` | Topic presence (0/1) |
| `dds_topic_inconsistent_total` | Topic inconsistency events |

#### Rollup (computed by recording rules in `prometheus/monitoring_recording_rules.yml`)
| Prometheus metric | Description |
|---|---|
| `dds_errors` | System error rollup (0 = OK, 1 = error) |
| `dds_data_loss_errors` | Data loss rollup |
| `dds_delays_errors` | Deadline delay rollup |
| `dds_saturation_errors` | Saturation rollup |
| `dds_excessive_bandwidth_errors` | Bandwidth rollup |

---

### UNVERIFIED Metric Names (DO NOT USE in PromQL until confirmed)

The following metric names are **expected** based on DDS API semantics but could
**not be verified** against the Connext 7.3.1 Prometheus export catalog. Do not
add them to dashboards or recording rules until confirmed.

| Expected Prometheus name | Likely source | How to confirm |
|---|---|---|
| `dds_data_writer_matched_subscriptions_count` | DDS_PUBLICATION_MATCHED_STATUS | `curl :19090/metrics \| grep matched` |
| `dds_data_writer_matched_subscriptions_current_count_change` | As above | As above |
| `dds_data_reader_matched_publications_count` | DDS_SUBSCRIPTION_MATCHED_STATUS | As above |
| `dds_data_reader_matched_publications_current_count_change` | As above | As above |
| `dds_domain_participant_discovered_participants_count` | Discovery data | As above |
| `dds_domain_participant_discovered_publications_count` | Discovery data | As above |
| `dds_domain_participant_discovered_subscriptions_count` | Discovery data | As above |

---

### UNVERIFIED Internal `enabled_metrics_selection` Names

The values inside `<enabled_metrics_selection>` in `MonitoringProfiles.xml` are
**internal Connext monitoring metric IDs** — not Prometheus names. The complete
7.3.1 metric catalog is not exposed in the public RTI documentation.

The profiles currently use category-level fnmatch patterns (e.g.,
`dds_data_writer_protocol_*`) which are more targeted than a bare `*` but still
wildcard-based. Replacing these with exact IDs would require scraping a running
Collector Service:

```bash
# With Collector Service running on localhost:19090:
curl -s http://localhost:19090/metrics | grep '^dds_' | cut -d'{' -f1 | sort -u
```

The resulting `dds_*` names are the Prometheus export names. The internal metric
IDs are likely similar (without the `_total` / `_mean` suffix added on export).

---

## Open Questions Summary

| # | Question | Blocking? | Action |
|---|---|---|---|
| OQ-1 | Does `DDS_MonitoringDistributionSettings` have a `participant_qos_profile_name` field in 7.3.1? | No — `MonitoringTransportBase` still compiles and the snippet still works | Check 7.3.1 API ref for `DDS_MonitoringDistributionSettings` |
| OQ-2 | Are `WARNING`, `STATUS`, `LOCAL` valid values for `middleware_forwarding_level`? | No — worst case the XML will fail to load | Check `DDS_NDDS_APPLICATION_LOG_VERBOSITY_KIND` enum in 7.3.1 API |
| OQ-3 | Does `event_settings.publication_period = DDS_DURATION_ZERO` mean "immediate" or "disabled"? | No — only affects event latency | Confirm from MONITORING QosPolicy docs or empirical test |
| OQ-4 | Do match/subscription-count metrics exist in the Prometheus export (e.g., `dds_data_writer_matched_subscriptions_count`)? | No — Discovery dashboard functions without them | Scrape live Collector Service and `grep matched` |
| OQ-5 | Do participant discovery-count metrics exist (discovered_participants, discovered_publications, discovered_subscriptions)? | No — Discovery dashboard functions without them | Scrape live Collector Service |
| OQ-6 | What are the exact internal `enabled_metrics_selection` IDs? | No — category wildcards work | Scrape Collector Service and cross-reference with metric catalog |

---

## Files Created

| File | Purpose |
|---|---|
| `qos/Monitoring.xml` | Tiered participant factory QoS profiles (Production, Development, Discovery) + dedicated participant config |
| `qos/Logging.xml` | Application logging configuration |
| `qos/Ping.xml` | DDS Ping test app QoS profiles |
| `grafana/dashboards/Production/production_dashboard.json` | Health, loss, delay, saturation |
| `grafana/dashboards/Development/development_dashboard.json` | Throughput, QoS, cache, topology |
| `grafana/dashboards/Discovery/discovery_dashboard.json` | Presence, churn, convergence |
| `docker-compose.yml` | Orchestration for Collector Service 7.7.0, Prometheus, Grafana |
| `docs/observability_tiers.md` | This document |

---

## Grafana Provisioning

The existing `grafana/dashboards/dashboard.yml` provisions dashboards recursively
from `/etc/grafana/provisioning/dashboards` (which maps to `grafana/dashboards/`).
The new `Production/`, `Development/`, and `Discovery/` subdirectories are
automatically discovered — no changes to `dashboard.yml` are needed.

---

## References

- RTI Connext 7.3.1 MONITORING QosPolicy: https://community.rti.com/static/documentation/connext-dds/7.3.1/doc/manuals/connext_dds_professional/users_manual/users_manual/MONITORING_QosPolicy.htm
- RTI Observability Framework: https://community.rti.com/static/documentation/connext-dds/7.3.1/doc/manuals/connext_dds_professional/users_manual/users_manual/ObservabilityLibrary.htm
- DDS_MonitoringMetricSelection: https://community.rti.com/static/documentation/connext-dds/7.3.1/doc/api/connext_dds/api_cpp/structDDS__MonitoringMetricSelection.html
- DDS_MonitoringDistributionSettings: https://community.rti.com/static/documentation/connext-dds/7.3.1/doc/api/connext_dds/api_cpp/structDDS__MonitoringDistributionSettings.html
- Builtin QoS Profiles: https://community.rti.com/static/documentation/connext-dds/7.3.1/doc/api/connext_dds/api_cpp/group__DDSBuiltinQosProfilesModule.html
- Collector Service Docker Hub: https://hub.docker.com/r/rticom/collector-service
