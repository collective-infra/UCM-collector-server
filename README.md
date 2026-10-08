# UCM-collector-server

This repo contains the configuration files for the Debian server that collects the sensor readings.

The firmware for the sensor is [in the ucm-node repo](https://github.com/jmarti23/ucm-node)

## UCM Sensor Node: Data Pipeline

Air-quality sensor nodes publish readings over MQTT. Telegraf subscribes to the
broker and writes into PostgreSQL withthe TimescaleDB extension, and Grafana visualizes the data.

```
ESP32 node ──MQTT──▶ Broker ──▶ Telegraf ──▶ PostgreSQL + TimescaleDB ──▶ Grafana
```

## MQTT topics

Broker: `mqtt://nyit-ucm.nycmesh.net` (port 1883). Every topic is namespaced per node as `ucm/node-<node_id>/...`.

| Topic | Retained | Rate | Payload |
|---|---|---|---|
| `ucm/node-<id>/status` | yes | on connect / LWT | `{"node","status":"online"\|"offline"}` |
| `ucm/node-<id>/info` | yes | once at boot | `{"node","name","description","lat","lon"}` |
| `ucm/node-<id>/environment` | no | ~1 s | `{"node","timestamp","pm1","pm25","pm4","pm10","temperature","humidity","voc"}` |
| `ucm/node-<id>/heartbeat` | no | ~1 s | `{"node","timestamp","uptime_s","free_heap","rssi"}` |
| `ucm/node-<id>/command` | no | n/a | Inbound commands (see below) |

Notes:

- `rssi` is `0` when the node has no Wi-Fi station info (e.g. on Ethernet); don't treat it as a real signal strength.
- Commands must match the payload exactly:
  - `{"command":"ota"}` downloads and installs the firmware from `http://nyit-ucm.nycmesh.net/ota/ucm-node.bin` (hardcoded in the firmware).

```bash
# Trigger OTA on one node. do NOT use -r (retain)
mosquitto_pub -h nyit-ucm.nycmesh.net -q 1 \
  -t 'ucm/node-<node_id>/command' -m '{"command":"ota"}'
```

```bash
# Subscribe to all topics. Used for monitoring
mosquitto_sub -h nyit-ucm.nycmesh.net -t '#' -v
```