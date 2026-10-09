# E5 Smart Car Parking SaaS Case Study (computer vision)
> Last verified: 2026-10 · Audience: senior/staff DevOps, SRE, AI eng, NetEng, DevSecOps, architects

## TL;DR
- Treat it as a **multi-tenant edge-AI SaaS**: thousands of RTSP cameras across hundreds of sites (car parks), two core ML tasks, **bay occupancy detection** and **ALPR/ANPR** (licence plate recognition) at entry/exit lanes, which feed billing, enforcement, guidance signs and dashboards.
- **Inference runs at the edge, not in the cloud.** Stream video to the cloud and you pay for about 3 Mbps per camera around the clock (about 32 GB per camera per day). Send events instead and it is about 1 KB per state change. Edge inference also keeps the site working through WAN outages and keeps raw plates and faces on site (privacy).
- Edge pipeline: **decode (HW) → detector (YOLO-family) → tracker (ByteTrack/DeepSORT/NvDCF) → plate localiser → OCR → temporal smoothing/debounce → event**. On NVIDIA hardware this is **DeepStream** (GStreamer + TensorRT). Elsewhere it is **ONNX Runtime** with an OpenVINO, QNN or TensorRT execution provider.
- Events travel over **MQTT (QoS 1)** with a store-and-forward spool and idempotent event IDs. In the cloud they go to an ingest stream, then an **occupancy state store** (latest state per bay), a time-series/lake for analytics, and APIs, webhooks and payments.
- Model lifecycle is a **closed loop**. The edge uploads low-confidence and disagreement samples (redacted), they get labelled, the model is retrained per region/plate-format, compiled per hardware target, then shipped **OTA with a per-site canary** and automatic rollback on KPI regression.
- Multi-tenancy: tenant → site → gateway → camera hierarchy, with **per-tenant topic namespaces and IoT policies**, per-tenant config (zones, bay polygons, tariffs, retention, plate-hashing salt) and per-tenant/region data residency.
- Cloud mapping: **AWS IoT Core + IoT Greengrass V2 + Kinesis Video Streams (on-demand clips) + SageMaker AI** ↔ **Azure IoT Hub + IoT Edge 1.6 LTS, or Azure IoT Operations on Arc-enabled K8s + Azure ML**.
- Know the retirements (2026): **AWS Panorama ended 2026-05-31**, **SageMaker Edge Manager ended 2024-04-26**, **Greengrass V1 ended 2026-10-07**, **AWS IoT Analytics discontinued 2025-12-15**, **Azure Percept DK retired 2023-03-30**, **Azure Video Analyzer retired 2022-12** (unverified exact day), **Lookout for Vision** end of support (announced 2024, unverified date: 2025-10-31). If you propose any of these in 2026, the interviewer will flag it.

## E5.1 Requirements Analysis

### Functional requirements
| # | Requirement | Notes / acceptance |
|---|---|---|
| F1 | **Bay-level occupancy** (free/occupied per bay) and zone/level counts | Bay polygons drawn per camera. One camera usually covers about 6–30 bays in an outdoor or overhead view |
| F2 | **ALPR at entry/exit lanes** | Plate string, country/region format, confidence, crop (kept local), timestamp, lane ID |
| F3 | **Session building**: entry plate → exit plate → duration → tariff → payment | Ticketless parking; supports permit and allow-list matching and barrier open |
| F4 | **Guidance**: real-time free-space counts to signs/apps | Low-latency path (local sign controller) |
| F5 | **Enforcement / overstay / wrong-bay (EV, disabled)** alerts | Alert with evidence only on violation |
| F6 | **Dashboards, APIs, webhooks** per tenant | Occupancy history, turnover, dwell time, revenue |
| F7 | **Fleet ops**: provision cameras/gateways, config, OTA, health | Zero-touch provisioning at site install |

### Non-functional requirements (state numbers explicitly)
- **Scale**: about 500 tenants, 2,000 sites, **10k–50k cameras**, 1–5 gateways per site. Event rates are low (a bay changes state roughly 10–30 times/day). Lane cameras run in bursts at peak (for example 8–9 am entry).
- **Latency**: barrier-open decision **< 1 s** from plate visible to gate signal, so it must be local. Sign update < 5 s. Cloud dashboard freshness < 10–30 s.
- **Accuracy KPIs**: occupancy per-bay accuracy **≥ 98–99%** after smoothing. ALPR **plate-level exact-match read rate ≥ 95–98%** in daylight. Track **character error rate** and **capture rate** separately (a plate that was never detected is a different failure from a misread).
- **Availability / offline tolerance**: the site must keep working (barriers, local signs, session building) through **hours to days** of WAN loss. Buffer events locally (size for ≥ 72 h; Azure IoT Operations documents 72 h max offline) and replay **in order and idempotently** on reconnect.
- **Bandwidth**: assume poor uplinks (4G/LTE, shared DSL). Budget **< 50–100 kbps steady per site** for telemetry, with bursts for clips/samples rate-limited.
- **Privacy / compliance**: plates are **personal data** under GDPR and similar laws, and faces/pedestrians are incidental biometrics-adjacent data. Requirements:
  - **Data minimisation**: no raw video leaves the site by default.
  - **Plate pseudonymisation**: keyed hash (HMAC with a per-tenant key held in KMS/Key Vault) for analytics. Clear-text plates only in the billing/enforcement bounded context with short retention.
  - **Redaction**: blur faces and non-target plates on any image used for training or evidence.
  - **Retention per tenant** (for example 30 days for sessions, 7 days for evidence crops unless disputed), **DSAR/erasure** support, **data residency** per tenant region.
- **Security**: per-device X.509 identity (TPM/secure element where possible), mTLS to the cloud, signed model and software artefacts, cameras on an isolated VLAN (cameras are notoriously insecure, so never expose RTSP to the internet).
- **Cost**: unit economics per camera per month. The edge box ($300–$2,000) is amortised. Cloud costs should be dominated by messaging and storage, not GPU inference.

### Back-of-envelope (say it out loud)
| Option | Per camera | 10k cameras |
|---|---|---|
| Stream 1080p H.264 at ~3 Mbps to cloud | 3 Mbps ≈ **32 GB/day** | 30 Gbps, **~320 TB/day** ingest + cloud GPU decode/inference |
| Upload 1 snapshot/min (~150 KB JPEG) | ~216 MB/day | ~2 TB/day, but occupancy goes stale and plates are missed |
| **Edge inference, events only** (~1 KB × ~500 events/day/camera incl. heartbeats) | **~0.5 MB/day** | **~5 GB/day**. Messaging-priced, trivially cheap |
| Plus active-learning samples (redacted crops, capped e.g. 50/camera/day × 50 KB) | ~2.5 MB/day | ~25 GB/day, scheduled off-peak |

- Edge compute sizing: a Jetson Orin-class module or small x86 + iGPU/NPU handles roughly **8–32 1080p streams** at reduced analysis FPS (unverified, depends on model and resolution). Occupancy needs only **1–2 FPS** (cars park slowly). ALPR lanes need **10–25 FPS** on the lane camera with a tight ROI.
- So **split by duty**: many cheap occupancy streams at low FPS plus a few high-FPS lane streams per gateway.

### Edge vs cloud inference trade-off
| Dimension | Edge inference (recommended) | Cloud inference (stream/snapshots) |
|---|---|---|
| Bandwidth | Events only (KB) | Mbps per camera, 24×7 |
| Latency (barrier) | Sub-second, local | WAN RTT + queueing; fails offline |
| Offline tolerance | Full local operation + spool | None |
| Privacy | Raw video stays on site | Plates/faces cross WAN and sit in cloud storage |
| Model iteration speed | Slower: OTA, heterogeneous hardware, compile per target | Fast: one deployment, elastic GPU |
| Fleet ops burden | High: thousands of boxes, disks, thermals, OS patching | Low |
| Accuracy ceiling | Limited by edge compute (smaller models, INT8) | Larger models, ensembles |
| Cost shape | Capex per site + modest cloud | Opex dominated by egress/ingest + GPU |

- **Hybrid** is the staff-level answer:
  - Edge does real-time inference.
  - The cloud does **second-opinion OCR** on low-confidence lane reads. Send only the plate crop, encrypted, a few KB. Cloud OCR can be a larger model or a managed text-detection API.
  - The cloud also does batch re-scoring, analytics and training.
- **Smart cameras** (on-camera NPU running the detector) remove the gateway, but you lose a uniform runtime. Use them only if the vendor SDK supports your OTA and model formats.

### Interview angles
- "Why not just send video to the cloud?" Do the bandwidth maths above, then cover offline barriers, privacy, and GPU cost in the cloud. Then concede what cloud inference does better: model iteration and fleet simplicity.
- "What is your accuracy metric?" Occupancy: per-bay accuracy **and transition precision/recall** (flicker creates false sessions). ALPR: **end-to-end session match rate** (entry plate == exit plate), because that is what billing depends on. A single-frame OCR score is the wrong metric.
- "How do you handle night/rain/glare/occlusion?" IR illumination for lane cameras, a shutter speed fast enough to avoid motion blur, **multi-frame voting** over a track (take the best-quality frames and do per-character majority vote), and occupancy temporal smoothing (state must hold for N seconds, with hysteresis).
- "What about privacy?" On-device redaction, hashing, residency, retention and DPIA. Plates are PII. Faces are not needed at all, so blur them before any frame leaves the box.
- Pitfalls:
  - Designing a cloud-only system.
  - Ignoring clock sync. Use NTP/PTP on gateways, because session duration equals money.
  - Missing **idempotency** on replayed events (duplicate billing).
  - Treating "camera online" as "camera useful". Camera tampering, defocus, a moved FOV or a lens covered by snow all need health checks.

## E5.2 Architecture Design

### End-to-end architecture
```mermaid
flowchart LR
  subgraph SITE["Customer site (per tenant)"]
    CAM1["Occupancy cams<br/>RTSP H.264/H.265"] --> DEC
    CAM2["Lane ALPR cams<br/>RTSP + IR"] --> DEC
    subgraph GW["Edge gateway (Greengrass / IoT Edge / AIO on K8s)"]
      DEC["HW decode<br/>NVDEC/VAAPI"] --> DET["Detector<br/>YOLO-family INT8/FP16"]
      DET --> TRK["Tracker<br/>ByteTrack / NvDCF"]
      TRK --> OCC["Bay polygon logic<br/>+ hysteresis"]
      TRK --> LPD["Plate detect + OCR<br/>multi-frame vote"]
      OCC --> EVT["Event builder<br/>event_id, seq, ts"]
      LPD --> EVT
      LPD --> BAR["Local barrier / sign<br/>controller"]
      EVT --> SPOOL["Local MQTT broker<br/>+ disk spool"]
      LPD -. "low-conf crops redacted" .-> SMP["Sample uploader<br/>rate-limited"]
      RING["Local ring buffer<br/>24-72h video"]
    end
  end
  SPOOL -- "MQTT over mTLS QoS1" --> HUB["IoT Core / IoT Hub / Event Grid MQTT"]
  HUB --> ING["Stream ingest<br/>Kinesis / Event Hubs / Kafka"]
  ING --> PROC["Stream processor<br/>dedupe + order + sessionise"]
  PROC --> STATE["Occupancy state store<br/>DynamoDB / Cosmos DB / Redis"]
  PROC --> TS["Time-series + lake<br/>Timestream / ADX / S3-ADLS Iceberg"]
  PROC --> SESS["Session + billing DB<br/>Postgres"]
  STATE --> API["Tenant APIs, WebSocket,<br/>webhooks"]
  SESS --> PAY["Payments / permits /<br/>enforcement"]
  TS --> DASH["Dashboards"]
  SMP --> LAKE["Training data lake"]
  RING -. "on-demand clip / dispute" .-> KVS["KVS / Blob clips"]
```

### Edge tier
- **Ingest**: cameras on an isolated VLAN, pulled via RTSP (prefer RTSPS where supported). Discover via **ONVIF**. Use the camera **substream** (lower resolution) for occupancy and the main stream with an ROI for ALPR.
- **Runtime choices**:
  - **NVIDIA DeepStream** (current 9.x supports Jetson Orin/Thor and dGPU). It is a GStreamer pipeline: `nvinfer` (TensorRT) for primary detection, `nvtracker`, secondary `nvinfer` for plate/OCR (cascaded), and `nvmsgbroker` to Kafka/MQTT/AMQP/Azure IoT. You get zero-copy NVDEC→inference. Python bindings are deprecated in favour of ServiceMaker.
  - **ONNX Runtime**: portable across hardware via execution providers (TensorRT/CUDA, OpenVINO for Intel iGPU/VPU, QNN for Qualcomm, Arm NN/ACL, XNNPACK). AWS's own guidance after Edge Manager EOL is **ONNX Runtime + Greengrass V2**.
  - **SageMaker Neo** (still documented) compiles models for edge targets (FP32/FP16/INT8). Treat it as optional. Most teams compile with TensorRT/OpenVINO directly.
- **Models**:
  - Occupancy uses either (a) a vehicle detector plus bay-polygon IoU, or (b) a per-bay crop classifier (cheaper and robust for fixed overhead cameras).
  - ALPR is three stages: vehicle detection → **plate detection** → **OCR** (CRNN/CTC or a transformer recogniser). Train per region, since plate syntax differs by country/state. Add a **syntax validator** (regex per jurisdiction) as a post-filter.
  - Typical quantisation is INT8 with a calibration set drawn from *each tenant's* camera domain.
- **Temporal logic**: an occupancy state changes only after N consecutive agreeing frames (for example 10 s), with hysteresis. For ALPR, the plate string is the per-character vote across the track's top-k sharpest frames, and the event is emitted when the track exits the ROI.
- **Event schema** (keep it small): `tenant_id, site_id, camera_id, bay_id|lane_id, type, state|plate_hash(+plate_enc), confidence, model_version, event_id (UUIDv7), seq, ts_capture`. Include `model_version` on every event, because that is how you attribute regressions.
- **Offline**: the local broker plus disk spool (Greengrass **stream manager** persists locally and exports to Kinesis Data Streams/S3 with a bandwidth cap; IoT Edge **edgeHub** store-and-forward; AIO MQTT broker). Local consumers (barrier, signs, session cache with permit list) keep working. On reconnect, replay in `seq` order and the cloud deduplicates on `event_id`.
- **Local video ring buffer** (24–72 h) for disputes. Upload a clip only on request. On AWS use the **KVS Edge Agent** (records locally, uploads on schedule or demand, ships as a Greengrass component). On Azure use the AIO **media connector** `clip-to-fs` writing to an Arc Container Storage cloud-ingest volume.

### Cloud tier
- **Device gateway**: AWS IoT Core (MQTT 128 KB payload cap, 100 publishes/s per connection, 512 KB/s per connection, 500k concurrent connections per account by default in big regions). Azure IoT Hub (D2C message ≤ 256 KB, 1M identities per hub, units scale throughput; S1 sends = max(100/s, 12/s/unit)). Azure Event Grid MQTT broker is an alternative for pure pub/sub.
- **Ingest → processing**: IoT rules/routes go to Kinesis Data Streams / Event Hubs / MSK/Kafka, partitioned by `site_id` (preserves per-site order). A stream processor (Flink / Kinesis Data Analytics successor *Managed Service for Apache Flink*, Azure Stream Analytics, or a K8s consumer) handles:
  - **dedupe by event_id** (TTL cache)
  - out-of-order handling using `ts_capture` + `seq`
  - **sessionisation** (entry↔exit plate match with fuzzy match, e.g. edit distance ≤ 1 on confusable characters O/0, B/8, then human review)
  - writes to the stores
- **Stores**:
  - **Current occupancy**: key-value store, key `tenant#site#bay` → state, ts, version, written with a conditional write (only if `ts` is newer). Hot reads go through Redis or a WebSocket fan-out.
  - **History/analytics**: time-series (Timestream for InfluxDB / Azure Data Explorer) plus an Iceberg/Delta lake for training and BI.
  - **Sessions/billing**: relational (Postgres) with strong consistency. Payments are idempotent with a request key.
- **APIs**: per-tenant REST/GraphQL, WebSocket for live maps, signed webhooks. Authorize using a tenant claim in the JWT.

### Model lifecycle (MLOps loop)
```mermaid
flowchart LR
  A["Edge: low-conf / disagreement /<br/>drift-triggered samples"] -->|"redact faces + plates not needed,<br/>encrypt, rate-limit"| B["Raw sample bucket<br/>per tenant prefix"]
  B --> C["Labeling<br/>Ground Truth / AML Data Labeling<br/>/ CVAT / Label Studio"]
  C --> D["Curated datasets<br/>versioned, region-tagged"]
  D --> E["Train<br/>SageMaker AI / Azure ML"]
  E --> F["Offline eval<br/>per-tenant golden sets,<br/>night/rain slices"]
  F -->|pass| G["Compile per target<br/>TensorRT / OpenVINO / ORT / Neo"]
  G --> H["Sign + register<br/>model registry"]
  H --> I["Shadow on canary site<br/>new model scores, old acts"]
  I --> J["Canary: 1 site -> 5% -> 25% -> 100%<br/>per tenant ring"]
  J -->|"KPI regression"| K["Auto rollback<br/>pinned previous version"]
  J --> L["Fleet"]
  L --> A
```
- **Data collection**: upload samples where confidence falls in a band (for example 0.3–0.7), where OCR fails the syntax validator, where entry/exit plates do not match, where occupancy flips more than N times/hour, plus a small **random sample** (unbiased estimate of accuracy). Gate all of it by **tenant consent/contract**.
- **Labeling**: SageMaker Ground Truth or Azure ML data labeling, or self-hosted CVAT/Label Studio. Use **pre-labelling** with the current model so humans correct rather than draw. Plates need exact transcription with double-keying QA.
- **Retraining cadence**: monthly baseline, plus triggers on drift (input stats like brightness histograms, detection count per frame, confidence distribution shift per camera).
- **Per-tenant specialisation**: one global backbone. Per-region OCR heads or fine-tunes. Avoid per-camera models (version explosion). Instead, keep **per-camera config** (ROI, bay polygons, thresholds) as data, not model.
- **OTA deployment**:
  - **AWS**: a new **Greengrass V2 component version** (model artefact in S3, recipe pins runtime). Deploy to **thing groups** (e.g. `tenant-A/canary`, `tenant-A/all`) via IoT Jobs-backed deployments with **rollout rate, timeout and abort criteria** (e.g. abort if > 5% fail). Rollback = redeploy previous component version. (SageMaker Edge Manager is gone; do not propose it.)
  - **Azure IoT Edge**: layered deployments targeting device-twin tags (`tags.ring='canary'`) with priority. Limit 100 deployments per hub, 50 modules per deployment. Models are delivered as a module image or downloaded blob referenced in module twin desired properties.
  - **Azure IoT Operations / Arc K8s**: GitOps (Flux) per cluster or Azure ML Kubernetes compute (Arc extension) for inference deployments. Use a canary via a cluster label selector.
- **Canary metrics** (compare against the control sites and against the site's own prior week):
  - occupancy flip rate
  - ALPR read rate and entry/exit match rate
  - inference latency p99 and FPS achieved
  - GPU temperature/throttling, memory
  - crash loops
- **Shadow mode** first for risky changes: the new model runs side by side and its results are logged but not acted on. This is cheap at the edge if there is GPU headroom.

### OTA canary sequence
```mermaid
sequenceDiagram
  participant CI as "CI/CD + Model registry"
  participant CP as "Fleet control plane<br/>(Greengrass deployments / IoT Hub layered deploy / GitOps)"
  participant GW as "Canary site gateway"
  participant MON as "Fleet monitoring"
  CI->>CP: Publish signed model vN+1 + component version
  CP->>GW: Deploy to ring canary-site
  GW->>GW: Verify signature, warm up TensorRT engine
  GW->>MON: Health + KPI telemetry tagged model_version
  MON-->>CP: KPIs within SLO for 24-72 h?
  alt regression or failure rate above abort threshold
    CP->>GW: Roll back to vN
  else healthy
    CP->>CP: Expand ring 5% then 25% then 100% per tenant
  end
```

### Multi-tenancy
| Concern | Design |
|---|---|
| Identity | One X.509 cert per gateway. Thing/device ID encodes `tenant/site/gw`. Fleet provisioning / DPS with enrollment groups **per tenant** |
| Topic isolation | Topics `t/{tenant}/s/{site}/...`. AWS IoT policy variables (`${iot:Connection.Thing.ThingName}`, thing attributes) restrict publish/subscribe to own prefix. Azure IoT Hub is per-device by design. Consider a **hub per tenant tier/region** |
| Data plane | Shared (pooled) streams + tenant_id partition/attribute for small tenants. **Silo** (dedicated account/subscription, KMS key, region) for enterprise/regulated tenants |
| Storage | Partition key starts with tenant. Per-tenant KMS/Key Vault keys for plate encryption and HMAC salt (crypto-shredding = tenant offboarding/erasure) |
| Config | Per-tenant/site config in a config service. Pushed via **device shadow / module twin desired properties** (ROI, bays, thresholds, retention, tariff ID). Versioned and audited |
| Noisy neighbour | Per-tenant quotas on API, webhook fan-out, sample uploads. IoT Hub/IoT Core account-level limits mean big tenants may need their own hub/account |
| Model customisation | Global model by default. Tenant-specific fine-tunes as a premium tier, using the same pipeline with a tenant-scoped dataset and **no cross-tenant training data without contract** |

### Fleet monitoring
- **Gateway telemetry**: CPU/GPU utilisation, temperature, throttling, disk (ring buffer fill), spool depth, uplink RTT/loss, NTP offset, component/module status (Greengrass system health telemetry to EventBridge; IoT Edge built-in metrics collector → Azure Monitor).
- **Camera health** (synthetic checks): stream up, FPS achieved, **blur/defocus score**, **scene-change/tamper** (compare to reference frame, detects a moved FOV), over/under-exposure, and "zero detections for N hours in a busy car park".
- **ML health per camera/model_version**: confidence histograms, flip rate, read/match rate, drift alarms.
- **SLOs**:
  - event freshness (ts_ingest − ts_capture p99)
  - site connectivity
  - occupancy accuracy (measured by periodic human audit samples)
  - ALPR match rate
- Alert on **SLO burn**, not on every device blip, because 10k devices make noise. Group by site/tenant.
- **Remote ops**: secure tunnel (AWS IoT secure tunneling / IoT Hub device streams) instead of opening inbound ports. Log shipping is sampled and rate-limited.

### Interview angles
- "How do you make the occupancy state correct with at-least-once delivery and offline replay?" Use idempotent event_id dedupe, per-bay monotonic `seq`/timestamp, and a **conditional write** (apply only if newer). Bay state is a CRDT-like last-writer-wins on capture time. The edge is the source of truth for the current state, and it **resyncs a full snapshot** on reconnect.
- "Barrier must open when cloud is down?" Yes. Use a local permit/allow-list cache, local session record, and deferred payment reconciliation. The policy decides whether unknown plates get a ticket, pay at kiosk, or open anyway (fail-open vs fail-closed is a **business decision**; safety egress usually fails open).
- "How do you roll out a bad model safely?" Shadow, then a site canary, then rings with abort thresholds, a pinned previous version, `model_version` on every event, and automatic rollback.
- "How do you keep plates private but still bill?" Split bounded contexts. Analytics see only `HMAC(tenant_key, plate)`. Billing holds encrypted plates with short retention. Erasure works by deleting by hash or by crypto-shredding the tenant key.
- "Heterogeneous hardware?" Use a hardware abstraction (ONNX + EP selection, or DeepStream only on NVIDIA SKUs), a compile matrix in CI per target, and a **small number of blessed SKUs**. Every new SKU multiplies the test matrix.
- Anti-patterns:
  - Per-camera models.
  - Raw video to the cloud "for training".
  - Long-lived shared credentials on gateways.
  - Opening RTSP to the internet.
  - Not pinning model + runtime versions together (a TensorRT engine is tied to the TensorRT version and GPU arch, so build engines on the device or per exact target).

## Cloud mapping: AWS vs Azure
| Capability | AWS | Azure | Role it plays | Key differences | Alternatives |
|---|---|---|---|---|---|
| Device connectivity / MQTT | **AWS IoT Core** | **Azure IoT Hub**; **Event Grid MQTT broker** | mTLS device auth, MQTT ingest, routing | IoT Core: 128 KB payload, per-account limits. IoT Hub: 256 KB D2C, units/tiers drive throughput, 1M identities per hub, max 50 hubs per subscription | EMQX/HiveMQ, Kafka + MQTT proxy |
| Edge runtime | **AWS IoT Greengrass V2** (components, nucleus / nucleus lite) | **Azure IoT Edge 1.6 LTS** (modules); **Azure IoT Operations** on **Arc-enabled K8s** | Deploy and run pipeline, local broker, store-and-forward | Greengrass: process/container/Lambda components on a Linux/Windows host. IoT Edge: Docker modules + edgeHub. AIO: K8s-native MQTT broker, data flows, Akri connectors, 72 h offline max | K3s + Flux GitOps, balena, NVIDIA Fleet Command (unverified current status) |
| Provisioning | IoT fleet provisioning (claim certs / JITP) | **Device Provisioning Service (DPS)**, Azure Device Registry (AIO) | Zero-touch per-tenant enrollment | DPS enrollment groups map cleanly to tenants | Manual CA + SCEP/EST |
| Device config / twin | Device Shadow | Device/module twin (32 KB desired/reported) | Push per-site config (ROI, bays) | Twin size limits. Shadow docs have a size limit (unverified: 8 KB per shadow doc, can be raised) | Config service + MQTT retained msgs |
| OTA / rollout | Greengrass deployments to thing groups (IoT Jobs rollout/abort) | IoT Edge layered deployments (100 per hub); GitOps on Arc | Canary and ring rollout of model + code | Both support targeted groups. Azure targets via twin tags + priority | Argo Rollouts / Flux on K3s, Mender |
| Local buffering to cloud | Greengrass **stream manager** (→ Kinesis Data Streams, S3, SiteWise; IoT Analytics target discontinued 2025-12-15) | edgeHub store-and-forward; AIO **data flows** → Event Hubs/Kafka, ADLS, Fabric | Offline spool + bandwidth cap | Stream manager has an explicit `EXPORTER_MAX_BANDWIDTH` | Kafka MirrorMaker / Redpanda edge |
| Video on demand / evidence | **Kinesis Video Streams** + **KVS Edge Agent** (local record, scheduled upload) | AIO **media connector** (snapshot-to-MQTT, clip-to-fs, stream-to-RTSP(S)) + Arc Container Storage → Blob | Dispute clips, labelling snapshots, live view | KVS is a managed time-indexed video store with HLS/DASH playback. Azure has no direct KVS equivalent since Video Analyzer retired, so clips go to Blob | Self-hosted MediaMTX/Frigate + object storage |
| Stream processing | Kinesis Data Streams / MSK + Managed Service for Apache Flink, Lambda | Event Hubs (Kafka API) + Stream Analytics / Functions | Dedupe, order, sessionise | Event Hubs has a Kafka endpoint, Kinesis is shard-based | Confluent/Kafka + Flink, Spark Structured Streaming / Databricks |
| State + history | DynamoDB, ElastiCache/MemoryDB, Timestream, S3 + Iceberg | Cosmos DB, Azure Managed Redis, Azure Data Explorer, ADLS + Delta / Fabric | Current occupancy, time-series, lake | Conditional writes in both (DynamoDB condition expressions / Cosmos ETag) | Postgres + TimescaleDB, ClickHouse |
| Training + labelling + registry | **SageMaker AI** (training, Ground Truth, Model Registry, Pipelines) | **Azure Machine Learning** (jobs, data labeling, registries, pipelines) | Retrain, eval, version | Azure ML can also deploy inference to **Arc K8s** (KubernetesCompute; CLI/SDK v1 retired) | Databricks/MLflow, CVAT, Label Studio |
| Edge model compile | **SageMaker Neo** (optional) | (none first-party; use ONNX Runtime / OpenVINO) | Hardware-specific optimisation | Neo is still documented. Edge Manager (signing/fleet) is gone | TensorRT, OpenVINO, ORT EPs, TVM |
| Managed CV APIs (cloud fallback) | Amazon Rekognition (text detection) | Azure AI Vision (OCR/Read) | Second-opinion OCR on crops | Generic OCR, not plate-tuned | Self-hosted OCR model, Gemini/Claude vision for review tooling (not real-time) |
| Remote access | IoT secure tunneling | IoT Hub device streams (limited, 50 concurrent) / Arc SSH | Break-glass ops | No inbound ports in either | Tailscale/WireGuard |

- **AWS role summary**: IoT Core is the device front door. Greengrass V2 runs the DeepStream/ONNX pipeline as components, with stream manager for offline export. KVS Edge Agent handles local recording plus on-demand clips. SageMaker AI is the training/labelling/registry loop, and Greengrass deployments do OTA.
- **Azure role summary**: there are two edge paths.
  - **(a) IoT Hub + IoT Edge 1.6 LTS.** Simple single-box gateways. 1.5 LTS support ends **2026-11-10**, so upgrade.
  - **(b) Azure IoT Operations on Arc-enabled K8s.** Multi-node sites, K8s-native, MQTT broker + data flows + media connector. Pairs with Event Grid MQTT/Event Hubs/Fabric in the cloud.
  - Azure ML covers training. It can target Arc K8s for on-prem inference, but for latency-critical video most teams still ship their own DeepStream/ONNX container.
- **Retired / do-not-propose (as of 2026-10)**:
  - **AWS Panorama**: end of support **2026-05-31**. It was exactly this use case (RTSP cameras + edge appliance), so expect a "what replaced it?" follow-up. Answer: Greengrass V2 + your own CV container on Jetson/x86.
  - **SageMaker Edge Manager**: EOL **2024-04-26**. AWS recommends ONNX Runtime + Greengrass V2, or MQTT + IoT Jobs for lightweight OTA.
  - **Greengrass V1**: end of support **2026-10-07**.
  - **AWS IoT Analytics**: discontinued **2025-12-15**.
  - **Amazon Lookout for Vision**: end of support announced Oct 2024 (unverified date: 2025-10-31).
  - **Azure Percept DK**: retired **2023-03-30**.
  - **Azure Video Analyzer**: retired Dec 2022 (unverified exact day).
  - **Azure ML CLI v1** ended 2025-09-30, SDK v1 2026-06-30. Use KubernetesCompute v2.
- **Alternatives**: a fully vendor-neutral edge (K3s + Flux + NVIDIA DeepStream + EMQX + Kafka/Confluent + Databricks for the lake/training) avoids hyperscaler lock-in. It is common for SaaS vendors selling into both AWS and Azure customers.

## Hands-on (optional)
Inspect a camera stream and benchmark edge throughput (bash/docker only):
```bash
# Probe an RTSP camera (codec, resolution, fps) from the gateway
ffprobe -v error -rtsp_transport tcp -select_streams v:0 \
  -show_entries stream=codec_name,width,height,avg_frame_rate \
  "rtsp://user:pass@10.20.0.11:554/stream2"

# Rough per-camera bitrate check (10 s sample) -> justifies events-not-video
ffmpeg -rtsp_transport tcp -i "rtsp://user:pass@10.20.0.11:554/stream1" -t 10 -c copy -f null - 2>&1 | grep -E "bitrate|speed"

# Watch the local event spool on the gateway's MQTT broker
mosquitto_sub -h localhost -p 8883 --cafile ca.pem --cert gw.pem --key gw.key \
  -t 't/tenantA/s/site42/#' -v
```
```yaml
# docker compose: minimal edge stack for a lab (not production)
services:
  broker:
    image: eclipse-mosquitto:2
    ports: ["1883:1883"]
    volumes: ["./mosquitto.conf:/mosquitto/config/mosquitto.conf"]
  deepstream:
    image: nvcr.io/nvidia/deepstream:9.1-triton-multiarch   # tag unverified; pick from NGC
    runtime: nvidia
    network_mode: host
    volumes: ["./ds-config:/opt/config", "./models:/opt/models"]
    command: ["deepstream-app", "-c", "/opt/config/parking_app.txt"]
    restart: unless-stopped
```

## Cross-links
- [E1 System design fundamentals](./E1-system-design-fundamentals.md): ML requirements framing, metrics, offline/online eval
- [E4 Content moderation case study](./E4-facebook-content-moderation-case-study.md): CV model selection and human-in-the-loop labelling
- [K4 LLM serving & inference](../K-ai-infra-llm/K4-llm-serving-inference.md): quantisation, batching, GPU utilisation concepts
- [K9 LLMOps, evals, guardrails](../K-ai-infra-llm/K9-llmops-evals-guardrails.md): canary/shadow evaluation patterns
- [M4 Kafka at scale](../M-data-platforms/M4-kafka-at-scale.md) and [M5 Stream processing](../M-data-platforms/M5-stream-processing.md): ingest partitioning, dedupe, out-of-order handling
- [L1 Data classification & PII](../L-data-privacy-ai-security/L1-data-classification-pii.md), [L3 Residency & compliance](../L-data-privacy-ai-security/L3-residency-compliance.md), [L2 Encryption & key management](../L-data-privacy-ai-security/L2-encryption-key-management.md): plates as PII, crypto-shredding
- [L7 Zero trust & workload identity](../L-data-privacy-ai-security/L7-zero-trust-workload-identity.md): device X.509 identity
- [C5 Deployment](../C-large-scale-architecture/C5-deployment.md): canary/ring rollouts
- [J2 Monitoring & alerting](../J-sre/J2-monitoring-and-alerting.md), [J1 SLIs/SLOs](../J-sre/J1-slis-slos-error-budgets.md): fleet SLOs and burn alerts
- [G9 Hybrid network basics](../G-cloud-network-architecture/G9-hybrid-network-basics.md): site uplinks, private connectivity

## Sources
- https://docs.aws.amazon.com/panorama/latest/dev/panorama-welcome.html (Panorama end of support 2026-05-31)
- https://docs.aws.amazon.com/sagemaker/latest/dg/edge.html and https://docs.aws.amazon.com/sagemaker/latest/dg/edge-eol.html (Edge Manager EOL 2024-04-26, ONNX + Greengrass guidance)
- https://docs.aws.amazon.com/sagemaker/latest/dg/neo.html (Neo)
- https://docs.aws.amazon.com/greengrass/v2/developerguide/what-is-iot-greengrass.html (Greengrass V1 EOS 2026-10-07)
- https://docs.aws.amazon.com/greengrass/v2/developerguide/stream-manager-component.html (stream manager, IoT Analytics discontinued 2025-12-15)
- https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/what-is-kinesis-video.html and https://docs.aws.amazon.com/kinesisvideostreams/latest/dg/edge.html (KVS, Edge Agent)
- https://docs.aws.amazon.com/general/latest/gr/iot-core.html (IoT Core quotas)
- https://learn.microsoft.com/en-us/azure/iot-edge/about-iot-edge (IoT Edge 1.6 LTS; 1.5 ends 2026-11-10)
- https://learn.microsoft.com/en-us/azure/iot-operations/overview-iot-operations (AIO, 72 h offline)
- https://learn.microsoft.com/en-us/azure/iot-operations/discover-manage-assets/howto-use-media-connector (media connector)
- https://learn.microsoft.com/en-us/azure/iot-hub/iot-hub-devguide-quotas-throttling (IoT Hub limits)
- https://learn.microsoft.com/en-us/azure/machine-learning/how-to-attach-kubernetes-anywhere (Azure ML on Arc K8s)
- https://learn.microsoft.com/en-us/previous-versions/azure/azure-percept/overview-azure-percept (Percept retirement)
- https://docs.nvidia.com/metropolis/deepstream/dev-guide/text/DS_Overview.html (DeepStream 9.1)
- https://onnxruntime.ai/docs/execution-providers/ (ONNX Runtime EPs)
