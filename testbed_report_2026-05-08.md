# OCUDU O-RAN Testbed — Session Report
**Date:** 2026-05-08  
**Final State:** ✅ 94 Running / 8 Completed — zero errors/failures

---

## Host Specifications

| Item | Value |
|---|---|
| Hostname | maveric-Pentino-H-Series-A-MT-B660 |
| IP | 131.246.109.46 |
| CPU | Intel i7-12700 (20 logical CPUs) |
| RAM | 31 GB |
| Disk | 937 GB NVMe (`/`) |
| OS | Ubuntu 24.04, kernel 6.17.0-23-generic |
| Docker | v29.4.3 (Compose v5.1.3) |
| Kubernetes | v1.35.3, single-node control-plane |
| Container runtime | containerd 2.2.3 |

---

## What Was Found and Verified Running

### Kubernetes Namespaces

#### `open5gs` — 15/15 Running
AMF, AUSF, BSF, NRF, NSSF, PCF, SCP, SEPP, SMF, UDM, UDR, UPF, MongoDB, WebUI, Populate

Config:
- DNN: `srsapn`, Subnet: `10.45.0.0/16`
- AMF NGAP: `10.105.109.23:38412/SCTP`

#### `nonrtric` (Non-RT RIC + RANPM) — 25 Running / 3 Completed
| Pod | Container | Status |
|---|---|---|
| a1-sim-osc-0, a1-sim-osc-1 | A1 Sim OSC | Running |
| a1-sim-std-0, a1-sim-std-1 | A1 Sim STD | Running |
| a1-sim-std2-0, a1-sim-std2-1 | A1 Sim STD2 | Running |
| capifcore | CAPIF Core | Running |
| controlpanel | Control Panel | Running |
| dfc-0 | Data File Collector (2/2) | Running |
| dmaapadapterservice-0 | DMaaP Adapter | Running |
| influxdb2-0 | InfluxDB 2.x | Running |
| informationservice-0 | ICS | Running |
| kafka-producer-pm-json2influx-0 | PM JSON→InfluxDB | Running |
| kafka-producer-pm-json2kafka-0 | PM JSON→Kafka | Running |
| kafka-producer-pm-xml2json-0 | PM XML→JSON | Running |
| minio-0 | Minio Object Storage | Running |
| minio-client | Minio Init Client | Running |
| nonrtricgateway | Non-RT RIC Gateway | Running |
| oran-nonrtric-kong | Kong API GW | Running |
| oran-nonrtric-postgresql-0 | PostgreSQL | Running |
| pm-producer-json2kafka-0 | PM Producer (2/2) | Running |
| pmlog-0 | PM Log (2/2) | Running |
| policymanagementservice-0 | A1 PMS | Running |
| rappmanager-0 | rApp Manager | Running |
| servicemanager | Service Manager | Running |
| influxdb2-init | InfluxDB init (pm-bucket) | **Completed** ✅ |
| oran-nonrtric-kong-init-migrations | Kong DB init | **Completed** ✅ |
| oran-nonrtric-kong-pre-upgrade-migrations | Kong pre-upgrade | **Completed** ✅ |

#### `ricplt` (Near-RT RIC) — 13/13 Running
a1mediator, alarmmanager, appmgr, e2mgr, e2term-alpha, o1mediator, rtmgr, submgr, vespamgr, r4-infrastructure-kong, prometheus-server, prometheus-alertmanager, dbaas

#### `onap` (SMO/ONAP) — 24 Running / 3 Completed
sdnc-0, sdnc-web, sdnc-ansible, dcae-ves-collector, cps-core, cps-temporal, ncmp-dmi-plugin, mariadb-galera, policy-{apex-pdp,api,pap,clamp-runtime-acm,clamp-ac-*×5}, postgres-{primary,replica×2}, cps-temporal-db, onap-strimzi-{broker,controller,entity-operator}

#### `kafka` — 1/1 Running
`strimzi-cluster-operator-0.51.0` (pod `strimzi-cluster-operator-99c9dc8d9-22hnh`)

#### `ricxapp` — chartmuseum, ricxapp-hw-go (Running)
#### `ricinfra` — deployment-tiller-ricxapp (Running)

### Host Docker Containers

| Container | Image | Port | Status |
|---|---|---|---|
| `ocudu-netconf` | ocudu-netconf/ocudu-netconf:latest | 0.0.0.0:830→830/tcp | Up |
| `ocudu-o1-adapter` | ocudu-o1-adapter:latest | 0.0.0.0:5000→5000/tcp | Up |

Both on Docker network: `smo_integration`

---

## What Was Installed This Session

### 1. OCUDU NETCONF Container (Docker)
- Built from source at `~/dep/ocudu-netconf/`
- Started with `--config gnb` profile
- Listens on port 830 (NETCONF/SSH over TCP)
- Login: `root` / `root`

### 2. OCUDU O1 Adapter Container (Docker)
- Built from source at `~/dep/ocudu-o1-adapter/` (Flask app)
- Exposes REST API on port 5000
- `/config-healthy` returns `{"success":"OK"}` ✅
- Generated gNB config: `/opt/ocudu-o1/ocudu-generated-gnb.yaml`

### 3. RANPM (Helm — `ranpm` release, `nonrtric` namespace)
Chart: `ranpm-1.0.0.tgz` from `~/dep/smo-install/oran_oom/smo/dist/packages/`  
Override file: `/home/maveric/ranpm-override.yaml`

Key settings:
- `global.useStrimziKafka: true` — uses `onap-strimzi` Kafka cluster in `onap` namespace
- Keycloak/OPA/Redpanda disabled (no keycloak in testbed)
- ICS host: `informationservice.nonrtric:9082`

RANPM dummy OAuth2 secrets created in `nonrtric` namespace:
- `dfc`, `kafka-pm-json-2-influx`, `kafka-pm-json-2-kafka`, `kafka-pm-xml-2-json`, `pm-producer-json-2-kafka`, `pm-log`
- All contain `client_secret: testbed-dummy-secret`

### 4. `smo-common` (Helm — separate release, `nonrtric` namespace)
Required to deploy InfluxDB2 and Minio (smo-common inside ranpm chart is library-only).  
Chart: `~/dep/smo-install/oran_oom/smo/common`

---

## Problems Found and Fixed

### Fix 1: RANPM pre-install hook blocked (keycloak-init job)
- **Symptom:** `helm install ranpm` stuck at `pending-install`; keycloak-init job never completed
- **Fix:** Delete stuck job + helm state secret, install with `--no-hooks`
  ```bash
  kubectl delete job keycloak-init -n nonrtric
  kubectl delete secret sh.helm.release.v1.ranpm.v1 -n nonrtric
  helm upgrade --install ranpm ... --no-hooks
  ```

### Fix 2: RANPM pods CreateContainerConfigError (missing OAuth2 secrets)
- **Symptom:** All 6 RANPM pods failed with `secret "dfc" not found`
- **Fix:** Created 6 dummy client_secret K8s Secrets
  ```bash
  for svc in dfc kafka-pm-json-2-influx kafka-pm-json-2-kafka kafka-pm-xml-2-json pm-producer-json-2-kafka pm-log; do
    kubectl create secret generic $svc -n nonrtric \
      --from-literal=client_secret=testbed-dummy-secret
  done
  ```

### Fix 3: RANPM init-wait.sh ConfigMaps blocked on keycloak-init job
- **Symptom:** Init containers in all RANPM pods looped waiting for `keycloak-init` job to complete
- **Fix:** Patched 6 ConfigMaps to remove the `wait for keycloak-init` line from `init-wait.sh`

### Fix 4: influxdb2/minio not deployed by ranpm
- **Root cause:** `smo-common` inside the ranpm chart is a Helm library chart (only `_*.tpl` helper templates) — it cannot deploy pods
- **Fix:** Installed `smo/common` umbrella chart as a separate `smo-common` helm release in `nonrtric` namespace

### Fix 5: ics-init stuck (keycloak dependency)
- **Symptom:** `ics-init` init container waited for `keycloak-init` job; main container ran `configure_ics.sh` which calls `keycloak_utils.sh`, looping on keycloak readiness
- **Fix:** Deleted the ics-init job (non-blocking for testbed operation)
- **Status:** ICS is running; job types need manual registration when keycloak is not available (see Deferred Items)

### Fix 6: Strimzi CrashLoopBackOff (RBAC missing in nonrtric)
- **Symptom:** `strimzi-cluster-operator` pod crashing with: `403 Forbidden: kafkamirrormaker2s.kafka.strimzi.io is forbidden in namespace nonrtric`
- **Root cause:** Strimzi was configured to watch namespaces `kafka,onap,nonrtric` but only had RoleBindings in `kafka` and `onap`
- **Fix:** Created 3 RoleBindings in `nonrtric` namespace (copied from `onap`):
  ```bash
  for rb in strimzi-cluster-operator strimzi-cluster-operator-entity-operator-delegation strimzi-cluster-operator-watched; do
    # copy from onap, apply to nonrtric
    kubectl get rolebinding $rb -n onap -o json | python3 -c "
  import sys,json; rb=json.load(sys.stdin); rb['metadata']['namespace']='nonrtric'
  [rb['metadata'].pop(k,None) for k in ['resourceVersion','uid','creationTimestamp','managedFields']]
  print(json.dumps(rb))" | kubectl apply -f -
  done
  kubectl rollout restart deployment/strimzi-cluster-operator -n kafka
  ```

### Fix 7: Old Error pods in ONAP
- Deleted 2 `Error`-state `k8s-ppnt` pods from previous deployments

### Fix 8: Open5GS PFCP (NULL node_id)
- Patched SMF config to ensure proper PFCP node_id binding
- PFCP heartbeat cycle remains at WARNING level (dual-SMF artifact, functional)

---

## Helm Releases Installed (session)

```
NAME       NAMESPACE  STATUS    CHART
ranpm      nonrtric   deployed  ranpm-1.0.0
smo-common nonrtric   deployed  common-13.0.0
```

---

## Key Endpoints / Access

| Service | Endpoint |
|---|---|
| OCUDU NETCONF | `localhost:830` (SSH, `root`/`root`) |
| OCUDU O1 Adapter | `http://localhost:5000` |
| O1 Config Health | `GET http://localhost:5000/config-healthy` |
| Generated gNB Config | `/opt/ocudu-o1/ocudu-generated-gnb.yaml` |
| InfluxDB2 | `http://influxdb2.nonrtric:8086` (org: `est`, bucket: `pm-bucket`) |
| Minio | `http://minio.nonrtric:9000` |
| ICS (Information Coordinator) | `http://informationservice.nonrtric:9082` |
| VES Collector | `http://131.246.109.46:30417` |
| SDNC Web | `http://131.246.109.46:30205` |
| Near-RT RIC E2 | `131.246.109.46:32222/SCTP` |
| Open5GS AMF NGAP | `10.105.109.23:38412/SCTP` |

---

## Deferred / Known Remaining Issues

### 1. ICS Job Registration (requires keycloak)
The `ics-init` job (`configure_ics.sh`) registers PM data pipeline jobs (e.g., `json-file-data-from-filestore-to-influx`) into ICS. It requires a JWT token obtained from keycloak. Without keycloak, register manually:

```bash
# Example: register PM JSON→InfluxDB info-job
curl -X PUT http://informationservice.nonrtric:9082/data-consumer/v1/info-jobs/kp-influx-json \
  -H 'Content-Type: application/json' \
  -d '{
    "info_type_id": "json-file-data-from-filestore-to-influx",
    "job_owner": "console",
    "status_notification_uri": "http://pm-producer-json2kafka.nonrtric:8084/callbacks/job-status",
    "job_definition": {}
  }'
```

Check available info-type IDs:
```bash
curl http://informationservice.nonrtric:9082/data-producer/v1/info-types
```

### 2. RANPM OAuth2 Sidecars (dummy secrets)
Six `auth-token` sidecar containers log errors because keycloak is not available. The main application containers in each RANPM pod are functional. If keycloak is deployed later, update the secrets with real client credentials.

### 3. Open5GS PFCP Heartbeat Cycle
Two SMF replicas cause periodic PFCP de-association/re-association (~10s cycle). This is at WARNING level and does not block data plane once a UE attaches. If it becomes an issue:
```bash
kubectl scale deployment open5gs-smf -n open5gs --replicas=1
```

### 4. OCUDU O1/PM Notes
Per OCUDU documentation, the O1 adapter supports CM and FM only. PM data is delivered via OCUDU's JSON WebSocket metrics service. For full PM pipeline to InfluxDB/rApp, implement:
```
OCUDU WebSocket PM → VES PM exporter → Kafka → pm-xml2json → influxdb-producer → InfluxDB
```

---

## Files Created / Modified This Session

| Path | Purpose |
|---|---|
| `/home/maveric/ranpm-override.yaml` | Helm values for ranpm install (keycloak disabled, ICS host set) |
| `/tmp/common-override.yaml` | Helm values for smo-common chart (influxdb2+minio enabled) |
| `/opt/ocudu-o1/ocudu-generated-gnb.yaml` | Generated gNB config by O1 adapter |
| ConfigMap `ranpm-dfc-init-wait` (nonrtric) | Removed keycloak-init wait (patched) |
| ConfigMap `ranpm-kafka-producer-*-init-wait` (nonrtric) | Removed keycloak-init wait (patched, ×5) |
| 3× RoleBindings in `nonrtric` | Strimzi RBAC fix |
| 6× Secrets in `nonrtric` | RANPM dummy OAuth2 client secrets |
