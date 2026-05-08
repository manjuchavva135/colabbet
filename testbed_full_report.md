# O-RAN 5G Testbed — Full Setup Report
**Date:** 2026-05-08  
**Node:** maveric-Pentino-H-Series-A-MT-B660 | IP: 131.246.109.46  
**Cluster state:** 94 Running / 8 Completed / 0 errors

---

## 1. Hardware & Platform

| Item | Value |
|---|---|
| CPU | Intel i7-12700, 20 logical CPUs |
| RAM | 31 GB |
| Disk | 937 GB NVMe, single mount `/` |
| OS | Ubuntu 24.04 LTS, kernel 6.17.0-23-generic |
| Docker | v29.4.3 (Compose v5.1.3) |
| Kubernetes | v1.35.3, single-node control-plane |
| Container runtime | containerd 2.2.3 |

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     SMO / Non-RT RIC Layer                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │
│  │  ONAP    │  │Non-RT RIC│  │  RANPM   │  │  InfluxDB2     │  │
│  │ (onap ns)│  │(nonrtric)│  │ Pipeline │  │  + Minio       │  │
│  │ VES/SDNC │  │ A1-PMS   │  │ DFC/PM   │  │  (nonrtric ns) │  │
│  │ CPS/NCMP │  │ ICS      │  │ Producers│  │                │  │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────┘  │
│          ▲ O1 / VES             ▲ A1-P               ▲ PM data  │
├──────────┼─────────────────────┼────────────────────────────────┤
│          │         Near-RT RIC Layer (ricplt ns)                 │
│  ┌───────┴──────────────────────────────────────────────────┐   │
│  │  E2-Term  │  E2-Mgr  │  A1-Mediator  │  O1-Mediator     │   │
│  │  SubMgr   │  AppMgr  │  VesPaMgr     │  RtMgr           │   │
│  └───────────────────────┬───────────────────────────────────┘  │
│                          │ E2AP (SCTP)                           │
├──────────────────────────┼──────────────────────────────────────┤
│                          │         RAN Layer (Docker host)       │
│  ┌───────────────────────┴──────────────────────────────────┐   │
│  │  OCUDU (simulated O-CU/O-DU)                             │   │
│  │  - NETCONF server :830 (CM/FM via O1)                    │   │
│  │  - O1 Adapter :5000 (REST→NETCONF bridge)                │   │
│  └───────────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│                    5G Core (open5gs ns)                          │
│  AMF | SMF | UPF | NRF | AUSF | UDM | UDR | PCF | NSSF | BSF  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. Component Inventory

### 3.1 Open5GS — 5G Core (`open5gs` namespace)

| Pod | Function | Status |
|---|---|---|
| open5gs-amf | Access & Mobility Function | ✅ Running |
| open5gs-smf | Session Management Function | ✅ Running |
| open5gs-upf | User Plane Function | ✅ Running |
| open5gs-nrf | Network Repository Function | ✅ Running |
| open5gs-ausf | Authentication Server Function | ✅ Running |
| open5gs-udm | Unified Data Management | ✅ Running |
| open5gs-udr | Unified Data Repository | ✅ Running |
| open5gs-pcf | Policy Control Function | ✅ Running |
| open5gs-nssf | Network Slice Selection Function | ✅ Running |
| open5gs-bsf | Binding Support Function | ✅ Running |
| open5gs-scp | Service Communication Proxy | ✅ Running |
| open5gs-sepp | Security Edge Protection Proxy | ✅ Running |
| open5gs-webui | Subscriber Management UI | ✅ Running |
| open5gs-mongodb | Core database | ✅ Running |
| open5gs-populate | Subscriber provisioning job | ✅ Running |

**Configuration:**
- DNN: `srsapn`
- UE subnet: `10.45.0.0/16`
- AMF NGAP address: `10.105.109.23:38412/SCTP`
- Helm: `open5gs-2.3.4` (Open5GS 2.7.5), revision 2

### 3.2 Near-RT RIC (`ricplt` namespace)

| Pod | Function | Status |
|---|---|---|
| deployment-ricplt-e2term-alpha | E2 Termination point | ✅ Running |
| deployment-ricplt-e2mgr | E2 Manager | ✅ Running |
| deployment-ricplt-a1mediator | A1 Interface Mediator | ✅ Running |
| deployment-ricplt-o1mediator | O1 Interface Mediator | ✅ Running |
| deployment-ricplt-submgr | Subscription Manager | ✅ Running |
| deployment-ricplt-rtmgr | Routing Manager | ✅ Running |
| deployment-ricplt-appmgr | xApp Manager | ✅ Running |
| deployment-ricplt-alarmmanager | Alarm Manager | ✅ Running |
| deployment-ricplt-vespamgr | VES Performance Mgmt | ✅ Running |
| statefulset-ricplt-dbaas-server-0 | Redis database | ✅ Running |
| r4-infrastructure-kong | API Gateway | ✅ Running |
| r4-infrastructure-prometheus-server | Prometheus metrics | ✅ Running |
| r4-infrastructure-prometheus-alertmanager | Alert manager | ✅ Running |

**Version:** `nexus3.o-ran-sc.org:10002/o-ran-sc/ric-plt-e2:6.0.7`  
**Helm:** 11 separate r4-* charts deployed

### 3.3 Non-RT RIC + RANPM (`nonrtric` namespace)

| Pod | Function | Status |
|---|---|---|
| policymanagementservice-0 | A1 Policy Management Service | ✅ Running |
| informationservice-0 | Information Coordinator (ICS) | ✅ Running |
| controlpanel | Non-RT RIC Control Panel UI | ✅ Running |
| nonrtricgateway | Non-RT RIC API Gateway | ✅ Running |
| rappmanager-0 | rApp Lifecycle Manager | ✅ Running |
| servicemanager | Service Discovery | ✅ Running |
| capifcore | CAPIF Core Function | ✅ Running |
| dmaapadapterservice-0 | DMaaP Adapter (ICS producer) | ✅ Running |
| a1-sim-osc-0/1 | A1 Policy Sim (OSC profile) | ✅ Running |
| a1-sim-std-0/1 | A1 Policy Sim (STD profile) | ✅ Running |
| a1-sim-std2-0/1 | A1 Policy Sim (STD2 profile) | ✅ Running |
| oran-nonrtric-postgresql-0 | PostgreSQL (Kong/rApps) | ✅ Running |
| oran-nonrtric-kong | Kong API Gateway | ✅ Running |
| **RANPM pipeline** | | |
| dfc-0 (2/2) | Data File Collector | ✅ Running |
| kafka-producer-pm-xml2json-0 | PM XML → JSON converter | ✅ Running |
| kafka-producer-pm-json2kafka-0 | PM JSON → Kafka forwarder | ✅ Running |
| kafka-producer-pm-json2influx-0 | PM JSON → InfluxDB writer | ✅ Running |
| pm-producer-json2kafka-0 (2/2) | PM file ICS producer | ✅ Running |
| pmlog-0 (2/2) | PM Log collector | ✅ Running |
| **Storage** | | |
| influxdb2-0 | Time-series DB (PM metrics) | ✅ Running |
| minio-0 | Object storage (PM files) | ✅ Running |
| minio-client | Minio init client | ✅ Running |
| influxdb2-init | Bucket init job | ✅ Completed |

**Helm:**
- `ranpm-1.0.0` (deployed 2026-05-08)
- `smo-common / common-1.0.0` (deployed 2026-05-08, provides influxdb2+minio)
- `oran-nonrtric / nonrtric-1.0.1` (helm status: failed — all pods Running despite this)

### 3.4 SMO / ONAP (`onap` namespace)

| Pod | Function | Status |
|---|---|---|
| onap-dcae-ves-collector | VES event collector | ✅ Running |
| onap-sdnc-0 | SDNC (CM via NETCONF) | ✅ Running |
| onap-sdnc-web | SDNC Web UI | ✅ Running |
| onap-sdnc-ansible-server | Ansible Server | ✅ Running |
| onap-cps-core | Configuration Persistence Service | ✅ Running |
| onap-cps-temporal | CPS Temporal | ✅ Running |
| onap-ncmp-dmi-plugin | NCMP DMI Plugin | ✅ Running |
| onap-policy-api | Policy API | ✅ Running |
| onap-policy-pap | Policy PAP | ✅ Running |
| onap-policy-apex-pdp | Policy APEX PDP | ✅ Running |
| onap-policy-clamp-runtime-acm | ACM Runtime | ✅ Running |
| onap-policy-clamp-ac-* (×5) | ACM Participants | ✅ Running |
| onap-strimzi-onap-strimzi-broker-0 | Kafka Broker | ✅ Running |
| onap-strimzi-onap-strimzi-controller-1 | Kafka Controller | ✅ Running |
| onap-strimzi-entity-operator | Kafka Entity Operator | ✅ Running |
| mariadb-galera-0 | MariaDB (SDNC/Policy) | ✅ Running |
| onap-postgres-primary/replica | PostgreSQL (CPS) | ✅ Running |
| onap-policy-postgres-primary/replica | PostgreSQL (Policy) | ✅ Running |
| onap-cps-temporal-db-0 | PostgreSQL (CPS temporal) | ✅ Running |

**Helm:** `onap-18.0.0` (Rabat release) — helm status `failed` but all pods are Running

### 3.5 Strimzi Kafka Operator (`kafka` namespace)

| Pod | Status |
|---|---|
| strimzi-cluster-operator-0.51.0 | ✅ Running |

Watches namespaces: `kafka`, `onap`, `nonrtric`

### 3.6 RAN Simulation — Docker Host

| Container | Image | Function | Status |
|---|---|---|---|
| `ocudu-netconf` | ocudu-netconf/ocudu-netconf:latest | NETCONF server (O-CU/O-DU CM/FM) | ✅ Up |
| `ocudu-o1-adapter` | ocudu-o1-adapter:latest | REST→NETCONF bridge (O1 adapter) | ✅ Up |

Network: `smo_integration` Docker bridge

### 3.7 xApps (`ricxapp` namespace)

| Pod | Status |
|---|---|
| chartmuseum | ✅ Running (Helm chart repo) |
| ricxapp-hw-go | ✅ Running (Hello World xApp) |

---

## 4. All Endpoints

### 4.1 NodePort — Externally Accessible (host IP: 131.246.109.46)

| Service | NodePort URL | Description |
|---|---|---|
| Open5GS WebUI | `http://131.246.109.46:30848` | Subscriber mgmt (default: admin/1423) |
| SDNC Web | `http://131.246.109.46:30205` | SDNC admin UI |
| SDNC RESTCONF | `http://131.246.109.46:30267` | RESTCONF API (8282→30267) |
| SDNC RESTCONF Alt | `http://131.246.109.46:31389` | RESTCONF Alt (8182→31389) |
| SDNC Call-Home | `tcp://131.246.109.46:30266` | NETCONF call-home |
| VES Collector | `http://131.246.109.46:30417` | VES event ingestion |
| Kafka External | `131.246.109.46:30493` | Strimzi external bootstrap |
| Kafka Broker-0 | `131.246.109.46:30490` | Direct broker access |
| Non-RT RIC Gateway | `http://131.246.109.46:30093` | Non-RT RIC API |
| Control Panel | `http://131.246.109.46:30091` | Non-RT RIC UI |
| Policy Mgmt Svc | `http://131.246.109.46:30094` | A1-PMS REST API |
| InfluxDB2 | `http://131.246.109.46:30759` | Time-series DB UI/API |
| Minio Web UI | `http://131.246.109.46:30712` | Object storage UI |
| Kong (Non-RT) | `http://131.246.109.46:32082` | Non-RT API proxy |
| Kong Admin (Non-RT) | `http://131.246.109.46:32081` | Kong admin |
| Kong (Near-RT) | `http://131.246.109.46:32080` | Near-RT API proxy |
| E2 Term (SCTP) | `sctp://131.246.109.46:32222` | E2AP interface (RAN→Near-RT RIC) |
| O1 Mediator NETCONF | `tcp://131.246.109.46:30830` | O1 NETCONF (Near-RT RIC) |
| Service Manager | `http://131.246.109.46:31575` | Service registration |

### 4.2 Docker Host Ports

| Service | URL | Description |
|---|---|---|
| OCUDU NETCONF | `ssh://root:root@localhost:830` | NETCONF/SSH (gnb profile) |
| OCUDU O1 Adapter | `http://localhost:5000` | O1 REST adapter |
| O1 Adapter Health | `GET http://localhost:5000/config-healthy` | Returns `{"success":"OK"}` |

### 4.3 Kubernetes Internal (ClusterIP)

| Service | ClusterIP:Port | Description |
|---|---|---|
| informationservice | `informationservice.nonrtric:9082` | ICS API |
| influxdb2 | `influxdb2.nonrtric:8086` | InfluxDB API (org: `est`, bucket: `pm-bucket`) |
| minio | `minio.nonrtric:9000` | S3 API |
| onap-strimzi-kafka-bootstrap | `onap-strimzi-kafka-bootstrap.onap:9092` | Kafka internal (SCRAM) |
| onap-sdnc | `sdnc.onap:8282` | SDNC internal RESTCONF |
| dcae-ves-collector | `dcae-ves-collector.onap:8080` | VES internal |
| open5gs-amf | `10.105.109.23:38412/SCTP` | AMF NGAP |

### 4.4 Interface Reference Table

| Interface | Standard | Endpoints | Status |
|---|---|---|---|
| N1/N2 (UE↔AMF) | 3GPP TS 38.413 | AMF NGAP `10.105.109.23:38412` | ✅ Ready (no UE attached) |
| N3 (UPF data plane) | GTP-U | UPF pod | ✅ Ready |
| N4 (SMF↔UPF) | PFCP | Internal K8s | ✅ Active (WARNING-level heartbeat) |
| E2 (gNB↔Near-RT RIC) | O-RAN E2AP v3 | `131.246.109.46:32222/SCTP` | ⚠️ **Missing gNB connection** |
| O1 CM (gNB→SDNC) | NETCONF | SDNC call-home `:30266`, OCUDU `:830` | ⚠️ **Not connected end-to-end** |
| O1 FM (gNB→VES) | VES | `131.246.109.46:30417` | ⚠️ **Not connected end-to-end** |
| O1 PM (gNB→VES/Minio) | VES/3GPP files | OCUDU WebSocket → VES exporter | ⚠️ **Missing VES PM exporter** |
| A1 (Non-RT→Near-RT RIC) | A1-AP | PMS `30094` ↔ A1-mediator | ✅ Sims running |
| R1 / ICS | O-RAN R1 | ICS `informationservice.nonrtric:9082` | ✅ Up (no jobs yet) |
| Kafka (PM pipeline) | Apache Kafka | `onap-strimzi:9092` (internal) | ✅ Active |

---

## 5. Kafka Topics & Users

### Topics (onap-strimzi cluster)

| Topic | Use |
|---|---|
| `collected-file-kt` (10p) | PM files collected by DFC |
| `json-file-ready-kp-kt` (10p) | JSON PM files ready (key-producer path) |
| `json-file-ready-kpadp-kt` (10p) | JSON PM files ready (adapter path) |
| `pmreports-kt` (10p) | Processed PM reports |
| `unauthenticated.sec-3gpp-performanceassurance-output-kt` | VES PM assurance |
| `unauthenticated.sec-3gpp-faultsupervision-output-kt` | VES fault |
| `unauthenticated.ves-measurment-output-kt` | VES measurement |
| `unauthenticated.sec-heartbeat-output-kt` | VES heartbeat |
| `cm-events-kt` | CPS CM events |
| `cps-data-updated-events-kt` | CPS data updates |
| `ncmp-events-kt` | NCMP events |
| `policy-pdp-pap`, `policy-heartbeat`, etc. | Policy framework |

### KafkaUsers (service accounts for RANPM)

| KafkaUser | RANPM Component |
|---|---|
| `service-account-dfc` | Data File Collector |
| `service-account-kafka-producer-pm-json2influx` | PM JSON→InfluxDB |
| `service-account-kafka-producer-pm-json2kafka` | PM JSON→Kafka |
| `service-account-kafka-producer-pm-xml2json` | PM XML→JSON |
| `service-account-nrt-pm-log` | PM Log |
| `service-account-pm-producer-json2kafka` | PM Producer |

---

## 6. Installed Helm Releases

| Release | Namespace | Chart | Status |
|---|---|---|---|
| `open5gs` | open5gs | open5gs-2.3.4 | ✅ deployed |
| `r4-*` (11 releases) | ricplt | r4-{component}-3.0.0 | ✅ deployed |
| `hw-go` | ricxapp | hw-go-1.0.0 | ✅ deployed |
| `strimzi-cluster-operator` | kafka | strimzi-kafka-operator-0.51.0 | ✅ deployed |
| `mariadb-operator[-crds]` | mariadb-operator | mariadb-operator-26.3.0 | ✅ deployed |
| `oran-nonrtric` | nonrtric | nonrtric-1.0.1 | ⚠️ failed* |
| `onap` | onap | onap-18.0.0 (Rabat) | ⚠️ failed* |
| `ranpm` | nonrtric | ranpm-1.0.0 | ✅ deployed |
| `smo-common` | nonrtric | common-1.0.0 | ✅ deployed |

\* Helm status `failed` means the initial helm install hit a timeout/hook error, but all pods are Running. The releases are functional.

---

## 7. ICS / R1 Interface State

The Information Coordinator Service (ICS) is Running at `informationservice.nonrtric:9082`.

| Item | State |
|---|---|
| Info-types registered | ❌ 0 (empty `[]`) |
| Info-producers registered | ✅ 1 (`dmaapadapterservice.nonrtric:9088`) |
| Info-jobs active | ❌ 0 (empty `[]`) |

**Why:** The `ics-init` job (`configure_ics.sh`) requires a keycloak JWT token to register the PM data pipeline job `json-file-data-from-filestore-to-influx`. Keycloak is not deployed in this testbed. The ics-init job was deleted (it looped indefinitely).

**To activate the PM pipeline manually:**
```bash
# 1. Check what info-types the pm-producer has published
curl http://10.105.13.251:9082/data-producer/v1/info-producers/https:__pm-producer-json2kafka.nonrtric:8084

# 2. Register the InfluxDB job
curl -X PUT http://10.105.13.251:9082/data-consumer/v1/info-jobs/kp-influx-json \
  -H 'Content-Type: application/json' \
  -d '{
    "info_type_id": "json-file-data-from-filestore-to-influx",
    "job_owner": "console",
    "status_notification_uri": "http://pm-producer-json2kafka.nonrtric:8084/callbacks/job-status",
    "job_definition": { "filter": "" }
  }'
```

---

## 8. Missing Interfaces for End-to-End 5G

This is the most important section. Below is what is **not yet connected** and what is needed for a complete E2E testbed.

### 8.1 ❌ No Real or Simulated gNB Connected

**Status:** No gNB (real or simulated) is sending E2AP to the Near-RT RIC.  
**Impact:** Near-RT RIC (E2-Term/E2-Mgr) is Running but idle. No xApps can receive RAN data.

**Options to connect a gNB:**

| Option | Tool | Notes |
|---|---|---|
| **Software gNB (recommended)** | srsRAN Project (`gnb` binary) | Full 5G SA, connects via E2AP to Near-RT RIC + NGAP to AMF. Needs USRP SDR or ZMQ virtual RF |
| **ZMQ-based gNB** | srsRAN + ZMQ | No hardware needed, virtual RF between gNB and UE |
| **ORAN-native gNB sim** | OSC O-DU High | Connects E2AP natively. Complex to build |
| **E2 simulator** | `e2sim` (O-RAN SC) | Simulates E2 node only, no real data plane |

**srsRAN gNB minimal config (ZMQ, no SDR):**
```yaml
# /etc/srsran/gnb.yaml
amf:
  addr: 10.105.109.23      # open5gs-amf ClusterIP
  port: 38412
e2:
  enable: true
  addr: 131.246.109.46     # E2-Term NodePort host
  port: 32222
  ran_node_id: 1
  plmn: "00101"
ru_sdr:
  device_driver: zmq
  device_args: "tx_port=tcp://127.0.0.1:2000,rx_port=tcp://127.0.0.1:2001,base_srate=11.52e6"
cell_cfg:
  dl_arfcn: 368500
  band: 3
  channel_bandwidth_MHz: 10
  nof_antennas_dl: 1
  nof_antennas_ul: 1
  plmn: "00101"
  tac: 7
```

### 8.2 ❌ No UE Attached

**Status:** No 5G UE (real or simulated) is registered.  
**Impact:** No data plane traffic, no bearer setup, no N1/N2/N3 flows.

**Options:**

| Option | Tool | Notes |
|---|---|---|
| **ZMQ UE (simplest)** | srsRAN `srsue` or `srs-ue` | Pairs with ZMQ gNB, no hardware needed |
| **UERANSIM** | UERANSIM (open-source) | Simulates 5G-SA UE, connects to AMF via NGAP |
| **Real UE + SDR** | Any 5G SA SIM + USRP | Production-like test |

**UERANSIM UE minimal config:**
```yaml
# ue.yaml
mcc: '001'
mnc: '01'
key: '465B5CE8B199B49FAA5F0A2EE238A6BC'  # must match Open5GS subscriber
op: 'E8ED289DEBA952E4283B54E88E6183CA'   # OP or OPc
opType: OP
imsi: '001010000000001'                   # must exist in Open5GS WebUI
apn: 'srsapn'
gnbSearchList:
  - 131.246.109.46   # or gNB address
```

**Add subscriber in Open5GS WebUI (`http://131.246.109.46:30848`):**
- IMSI: `001010000000001`
- Key: `465B5CE8B199B49FAA5F0A2EE238A6BC`
- OP: `E8ED289DEBA952E4283B54E88E6183CA`
- APN: `srsapn`
- SST: 1, SD: 000001

### 8.3 ❌ O1 CM Interface Not Connected End-to-End

**Status:** OCUDU NETCONF server is running on `:830`. SDNC is running. But SDNC does not know about the OCUDU device — no call-home or mount-point configured.

**What is needed:**

```
OCUDU NETCONF (:830) ←→ SDNC (NETCONF call-home or mount-point)
```

**Option A — NETCONF call-home (OCUDU initiates to SDNC):**
```bash
# Configure OCUDU to call home to SDNC
curl -X POST http://localhost:5000/configure \
  -H 'Content-Type: application/json' \
  -d '{
    "callhome": {
      "host": "131.246.109.46",
      "port": 30266
    }
  }'
```

**Option B — SDNC mounts OCUDU as a NETCONF node:**
```bash
# Create NETCONF mount point in SDNC via RESTCONF
curl -X PUT \
  http://131.246.109.46:30267/rests/data/network-topology:network-topology/topology=topology-netconf/node=ocudu-1 \
  -H 'Content-Type: application/json' \
  -u admin:Kp8bJ4SXszM0WXlhak3eHlcse2gAw84vaoGGmJvUy2U \
  -d '{
    "node": [{
      "node-id": "ocudu-1",
      "netconf-node-topology:host": "131.246.109.46",
      "netconf-node-topology:port": 830,
      "netconf-node-topology:username": "root",
      "netconf-node-topology:password": "root",
      "netconf-node-topology:tcp-only": false
    }]
  }'
```

After mounting, SDNC exposes the OCUDU config through NETCONF/RESTCONF and NCMP can manage it via CPS.

### 8.4 ❌ O1 FM / VES Not Connected

**Status:** VES Collector is running at `131.246.109.46:30417`. OCUDU O1 adapter is running. But no FM events are being sent.

**What is needed:** Configure OCUDU O1 adapter to send VES fault events:
```bash
curl -X POST http://localhost:5000/vesconfig \
  -H 'Content-Type: application/json' \
  -d '{
    "ves_endpoint": "http://131.246.109.46:30417/eventListener/v7",
    "ves_username": "sample1",
    "ves_password": "sample1"
  }'
```

VES Collector credentials (default): `sample1` / `sample1`

### 8.5 ❌ O1 PM / RANPM Pipeline Not Active (ICS Jobs Missing)

**Status:** RANPM is deployed and all pods Running. InfluxDB2 `pm-bucket` exists. Minio Running. But ICS has no registered jobs — no PM files will be collected/processed.

**Full PM data flow (what needs to happen):**
```
gNB/OCUDU
    │ PM XML files via SFTP/FTP or VES fileReady
    ▼
Minio (pm-files bucket)
    │ fileReady Kafka event → collected-file-kt
    ▼
DFC (Data File Collector)
    │ downloads PM file → puts in Minio → json-file-ready-kp-kt
    ▼
kafka-producer-pm-xml2json
    │ converts → json-file-ready-kpadp-kt
    ▼
kafka-producer-pm-json2influx
    │ writes metrics → InfluxDB2 (pm-bucket)
    ▼
InfluxDB2 (http://131.246.109.46:30759)
    │ Grafana / rApp query
```

**To activate:**
1. Register ICS job (see Section 7)
2. Upload a test PM XML file to Minio bucket `pm-files`:
```bash
# Port-forward Minio or use NodePort
kubectl port-forward -n nonrtric svc/minio 9000:9000 &
mc alias set local http://localhost:9000 minioadmin minioadmin
mc mb local/pm-files
mc cp test-pm-file.xml local/pm-files/
```
3. Send a `fileReady` VES event to trigger DFC:
```bash
curl -X POST http://131.246.109.46:30417/eventListener/v7 \
  -H 'Content-Type: application/json' \
  -u sample1:sample1 \
  -d '{
    "event": {
      "commonEventHeader": {
        "domain": "notification",
        "eventName": "Notification_RAN-Logical_fileReady",
        "sourceName": "ocudu-1",
        "reportingEntityName": "ocudu-1",
        "priority": "Normal",
        "version": "4.0",
        "vesEventListenerVersion": "7.0.1",
        "startEpochMicrosec": 1715000000000000,
        "lastEpochMicrosec": 1715000000000000,
        "sequence": 0
      },
      "notificationFields": {
        "notificationFieldsVersion": "2.0",
        "changeType": "FileReady",
        "arrayOfNamedHashMap": [{
          "name": "pm-file-1",
          "hashMap": {
            "location": "http://minio.nonrtric:9000/pm-files/test-pm-file.xml",
            "compression": "gzip",
            "fileFormatType": "org.3GPP.32.435#measCollec",
            "fileFormatVersion": "V10"
          }
        }]
      }
    }
  }'
```

### 8.6 ❌ No xApps Doing RAN Intelligence

**Status:** Only `hw-go` (Hello World) xApp is installed. This does nothing meaningful.

**What is needed for AI/ML-driven RAN:**

| xApp | Function | Source |
|---|---|---|
| KPI Monitor xApp | Reads E2 KPIs, exposes to Non-RT RIC | O-RAN SC cherry |
| QoE Predictor xApp | ML-based handover decisions | O-RAN SC cherry |
| Anomaly Detection xApp | PM data anomaly via A1 policy | Custom |
| RC (RAN Control) xApp | RAN slice control via E2 | O-RAN SC |

**Deploy an xApp via AppMgr:**
```bash
# List available xApps in chartmuseum
curl http://131.246.109.46:30525/api/charts | python3 -m json.tool

# Deploy via AppMgr API
curl -X POST http://131.246.109.46:32080/appmgr/ric/v1/xapps \
  -H 'Content-Type: application/json' \
  -d '{"xappName": "kpimon"}'
```

### 8.7 ❌ No A1 Policy Loop Active

**Status:** A1-PMS Running. A1-mediator Running. A1 simulators (6) Running. But no actual A1 policy is configured and no Near-RT RIC policy type is published.

**To create a policy type and instance:**
```bash
# Create policy type in A1-PMS
curl -X PUT http://131.246.109.46:30094/a1-policy/v2/policy-types/1 \
  -H 'Content-Type: application/json' \
  -d '{"name":"QoS","description":"QoS Control","policy_schema":{}}'

# Create policy instance
curl -X PUT http://131.246.109.46:30094/a1-policy/v2/policies/policy1 \
  -H 'Content-Type: application/json' \
  -d '{
    "ric_id": "ric1",
    "policy_type_id": "1",
    "policy_data": {"qos_target": 10}
  }'
```

---

## 9. End-to-End 5G Checklist

| # | Item | Status | Action Needed |
|---|---|---|---|
| 1 | Open5GS 5G Core | ✅ Running | — |
| 2 | Near-RT RIC | ✅ Running | — |
| 3 | Non-RT RIC + A1-PMS + ICS | ✅ Running | — |
| 4 | RANPM (DFC + producers) | ✅ Running | Register ICS jobs |
| 5 | InfluxDB2 (pm-bucket) | ✅ Running | — |
| 6 | Minio (object storage) | ✅ Running | Create `pm-files` bucket |
| 7 | Strimzi Kafka | ✅ Running | — |
| 8 | SDNC (CM management) | ✅ Running | Mount OCUDU device |
| 9 | VES Collector | ✅ Running | Configure OCUDU VES endpoint |
| 10 | OCUDU NETCONF server | ✅ Running (Docker :830) | Connect to SDNC |
| 11 | OCUDU O1 Adapter | ✅ Running (Docker :5000) | Configure VES/call-home |
| 12 | gNB (real or simulated) | ❌ **MISSING** | Deploy srsRAN/UERANSIM |
| 13 | 5G UE (real or simulated) | ❌ **MISSING** | Deploy srsRAN UE/UERANSIM |
| 14 | E2 interface (gNB→RIC) | ❌ **MISSING** | Needs gNB with E2AP |
| 15 | O1 CM connected | ❌ **MISSING** | SDNC mount-point for OCUDU |
| 16 | O1 FM/VES events | ❌ **MISSING** | Configure OCUDU VES output |
| 17 | O1 PM files flowing | ❌ **MISSING** | gNB PM + ICS job registration |
| 18 | ICS jobs registered | ❌ **MISSING** | Manual curl or deploy keycloak |
| 19 | xApp with E2 subscription | ❌ **MISSING** | Deploy KPImon or custom xApp |
| 20 | A1 policy loop | ❌ **MISSING** | Policy type + xApp A1 client |
| 21 | Keycloak / OAuth2 | ❌ Not deployed | Optional — needed for ICS auto-config |

---

## 10. Recommended Next Steps (Priority Order)

### Step 1 — Connect OCUDU to SDNC (O1 CM) — 30 min
Mount OCUDU NETCONF node in SDNC (see Section 8.3 Option B).  
Verify with: `curl http://131.246.109.46:30267/rests/data/network-topology:network-topology/topology=topology-netconf/node=ocudu-1`

### Step 2 — Register ICS Jobs (PM pipeline) — 10 min
See Section 7 curl command. Registers the JSON→InfluxDB pipeline without keycloak.

### Step 3 — Deploy srsRAN gNB with ZMQ (no hardware) — 2-4 hours
```bash
# Install srsRAN Project
sudo apt-get install cmake make gcc g++ pkg-config libfftw3-dev libmbedtls-dev \
  libsctp-dev libyaml-cpp-dev libgtest-dev libzmq3-dev
git clone https://github.com/srsran/srsRAN_Project.git
cd srsRAN_Project && mkdir build && cd build
cmake .. -DENABLE_EXPORT=ON -DENABLE_ZMQ=ON
make -j$(nproc)
```
Then configure `gnb.yaml` with AMF IP `10.105.109.23:38412` and E2 `131.246.109.46:32222`.

### Step 4 — Deploy UERANSIM (5G UE simulation) — 1 hour
```bash
git clone https://github.com/aligungr/UERANSIM
cd UERANSIM && make
./build/nr-ue -c config/open5gs-ue.yaml
```
Add subscriber in Open5GS WebUI first (IMSI, Key, OP, APN).

### Step 5 — Deploy KPImon xApp — 1 hour
Push KPImon chart to chartmuseum, deploy via AppMgr API. xApp subscribes to E2 KPM service model and publishes metrics to RAN data pipeline.

---

## 11. Known Issues / Deferred Items

| Issue | Severity | Notes |
|---|---|---|
| Helm `onap` status `failed` | Low | All pods Running. Helm fails on timeout, not deployment |
| Helm `oran-nonrtric` status `failed` | Low | All pods Running |
| ICS job registration needs keycloak | Medium | Manual curl workaround available (see Section 7) |
| RANPM OAuth2 sidecar errors | Low | `auth-token` sidecars log 401s — main app containers functional |
| PFCP heartbeat cycle (SMF/UPF) | Low | WARNING-level only. Caused by 2 SMF replicas. Scale to 1 to eliminate |
| `ics-init` job deleted | Medium | Manually register ICS jobs before activating PM pipeline |
| O1 PM via OCUDU | Medium | OCUDU O1 adapter supports CM+FM only; PM needs custom WebSocket exporter |
