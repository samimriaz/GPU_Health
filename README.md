# GPU Node Health Gates for Agent and Firmware Updates

## 1. Purpose and recommendation

Build a deployment safety service that answers:

> Did this update make GPU nodes, their communication fabric, or their data path unhealthy enough that we must stop the rollout?

The service compares pre-update and post-update evidence, checks absolute safety limits, and evaluates the updated nodes against unchanged controls. It sends a durable decision to the deployment orchestrator, which must enforce that decision before updating more nodes.

**Recommended design:**

- Keep the HSDU/HealthStore client on each bare-metal host for direct GPU, driver, runtime, network, and storage visibility.
- Send observations and urgent events to an independent, off-node HealthStore ingestion and decision service. Do not rely on the updated node to approve its own rollout.
- Add BMC and optional SoC/DPU observations as independent sources, not replacements for host telemetry.
- Stop on a single confirmed critical fault. For softer symptoms, require sustained regression plus corroboration, or a strong independently repeated end-to-end regression.
- Treat missing telemetry as `UNKNOWN`, never healthy. Hold promotion when required evidence is missing.
- Make the deployment unit topology-aware: GPU, node, distributed job, fabric partition, rack, and cluster are different failure scopes.
- Do not reduce the decision to one weighted health score. A good temperature must never cancel out an uncorrectable memory error.

This README is a proposed design, not an implemented service. Numeric thresholds below are initial policy candidates, **not NVIDIA or Azure guarantees**. Calibrate them in shadow mode for each hardware, firmware, workload, and network profile.

### Assumptions and terminology

- The initial target is Linux bare-metal NVIDIA GPU infrastructure, using Kubernetes or a comparable scheduler. Adapt the collectors and vendor-specific policies for AMD or other accelerators.
- "HSDU" means the existing node-side HealthStore client described in the request. Its expansion, APIs, privileges, and current delivery guarantees are not assumed.
- "SoC" is assumed to mean an independently running management processor or SmartNIC/DPU, such as a BlueField Arm subsystem, **not a security operations center**. Confirm this meaning before implementation.
- BMC is a management controller with firmware and out-of-band interfaces, not simply another host process.
- Host agent, NMAgent, and "NICE agent" are treated as examples from your environment. There is no assumed one-to-one mapping to a universal bare-metal GPU agent inventory.
- A node health failure is not proof the update caused it. Stop exposure first when necessary; determine attribution separately.

## 2. What gets updated on a GPU bare-metal node?

A bare-metal GPU server has the usual host-management stack **plus** GPU, high-speed fabric, and GPU-aware storage components. Inventory the actual installed services and firmware rather than relying on these example names.

| Layer | Common components or update targets | Main failure modes to gate |
|---|---|---|
| Provisioning and host management | Image/provisioning agent, lifecycle/configuration agent, security agent, telemetry client, certificate/configuration packages | Reboot loops, permission changes, CPU overhead, missing telemetry, broken device access |
| OS and platform | Kernel, GPU kernel modules, initramfs, BIOS/UEFI, CPU microcode, PCIe/IOMMU/NUMA settings | Driver incompatibility, PCIe errors, peer-to-peer path changes, placement regressions |
| GPU driver and firmware | NVIDIA driver, NVML, GSP firmware where applicable, device VBIOS/firmware through supported tooling | Missing GPUs, initialization/reset failures, errors, clock or performance changes |
| GPU fabric management | NVIDIA Fabric Manager on applicable NVSwitch systems; NVLink Subnet Manager (NVLSM) where required by the platform; NVSwitch firmware | Fabric not initialized, missing links, partition mismatch, failed collectives |
| Runtime and scheduler integration | Container runtime, NVIDIA Container Toolkit, Kubernetes device plugin, GPU Feature Discovery, MIG Manager where used; kubelet or Slurm components | GPUs visible to the host but unavailable to jobs; wrong resources or MIG layout |
| GPU monitoring and diagnostics | DCGM host engine, DCGM Exporter, health/diagnostic configuration | Missing fields, changed units, stale samples, observer-induced overhead |
| Host networking and RDMA | NIC driver/firmware, inbox RDMA or supported vendor stack, rdma-core, networking agents/CNI, Network Operator where used | RDMA failure, link downshift, retransmission, wrong routing or QoS |
| Fabric control plane | InfiniBand subnet management/UFM or Ethernet/RoCE switch OS, routing, QoS, ECN/PFC configuration | Shared congestion, partition/routing errors, multi-node communication failures |
| Optional SoC/DPU | DPU OS/BSP, DOCA components, virtual switch/offload/network services, DPU firmware and telemetry | Offload/routing failure, data-path stalls, host/DPU compatibility mismatch |
| Storage | NVMe firmware, storage initiator/client, filesystem modules, CSI components, GPUDirect Storage/cuFile and nvidia-fs where applicable | I/O stalls, read failures, CPU bounce-path fallback, checkpoint failures |
| Out-of-band hardware management | BMC firmware, sensor configuration, platform management bundles, retimer/CPLD firmware where applicable | Loss of management, power/cooling problems, PCIe training issues |
| Workload software | CUDA user-space libraries, NCCL, framework/container images, communication plugins | ABI incompatibility, collective hangs, numerical or performance regression |

Not all rows are "agents." Some are firmware, libraries, scheduler integrations, or shared fabric services. The same deployment gate should support all of them, but their drain, test, rollback, and failure-domain policies differ. GPU Operator manages several Kubernetes GPU components; it is not the management plane for every row [3].

Record a **version/configuration manifest** before and after each update: host image/kernel, target agent, driver, GPU firmware, Fabric Manager/NVLSM, NIC firmware/driver, DPU, storage client, CUDA/NCCL, runtime, GPU/MIG topology, power caps, and relevant fabric configuration. Include artifact digests and configuration hashes. Avoid changing multiple unrelated layers in one canary.

## 3. Health model: four independent dimensions

Evaluate these independently for each entity and aggregate without averaging away failures:

1. **Correctness and safety:** Can computations and I/O complete correctly without unrecoverable faults?
2. **Availability and topology:** Are all expected GPUs, links, ports, partitions, and runtime resources usable?
3. **Performance:** Does comparable work complete at an acceptable rate and latency?
4. **Observability:** Is the evidence fresh, complete, and trustworthy enough to decide?

Expose `PASS`, `WARN`, `FAIL`, and `UNKNOWN` for each dimension and domain. A final deployment decision is separate: `CONTINUE`, `HOLD`, or `HALT`.

- `CONTINUE`: all required gates pass and minimum observation/test coverage is complete.
- `HOLD`: insufficient evidence, ambiguous regression, or a potentially shared incident. Do not promote.
- `HALT`: a confirmed critical fault or qualifying regression. Fence further rollout and contain the affected scope.

An idle GPU can be healthy but unvalidated for performance. A node with no errors and no telemetry is not healthy.

## 4. Key metrics and supporting evidence

**Key metrics** directly establish safety, availability, correctness, or delivered performance. **Supporting metrics** explain a symptom and help distinguish update effects from workload or infrastructure noise.

Collect GPU-specific evidence per GPU UUID, MIG instance where applicable, link, and port. Also retain node-level and job-level views. Never gate on a cluster average alone.

### 4.1 Compute, memory, and GPU-local fabric

| Signal | Role | Source examples | How to use it |
|---|---|---|---|
| GPU presence, responsiveness, CUDA context/kernel success, expected schedulable resources | Key | NVML/DCGM, scheduler/device-plugin state, bounded runtime probe | Required GPU missing or unusable after the recovery deadline is a hard failure. Match UUIDs/topology, not enumeration indices. |
| New uncorrectable GPU memory errors; recovery-required or unrecoverable GPU state | Key | DCGM/NVML, driver events | A new confirmed uncorrectable error is a hard gate. Distinguish contained application impact, required recovery action, and node-wide impact. |
| Xid/SXid events and GPU recovery action | Key, classified | Driver event stream/kernel logs, NVML/DCGM | Use a versioned vendor-informed classification. Do not treat every nonzero Xid as a node failure or difference successive Xid codes as a counter [4]. |
| Memory row-remap failure, exhausted spare capacity, pending recovery/remap state | Key for failure; otherwise supporting | DCGM/NVML | Remapping failure is critical. Successful historical remaps are not automatically failures; pending states need SKU-specific maintenance policy. |
| Known-work compute throughput and HBM bandwidth; validation/checksum correctness | Key | Fixed compute/memory probe or matched workload instrumentation | Compare identical precision, shapes, batch, clocks/power policy, and placement. Incorrect output fails independently of speed. |
| Required NVLink/NVSwitch connectivity and fabric readiness | Key | NVML/DCGM, Fabric Manager/NVLSM, topology inventory | Loss of required connectivity after recovery is a hard availability failure. Disabled/unpopulated links are not failures. |
| Intra-node collective bandwidth/latency and correctness | Key | Controlled NCCL test or equivalent | Detect a server whose individual GPUs work but whose GPU-to-GPU fabric is degraded. |
| Correctable ECC rate and concentration | Supporting | DCGM/NVML | Detect bursts and concentration on one device; combine with memory/compute degradation or a classified memory-health warning. |
| SM activity, tensor activity, GPU utilization, HBM activity, memory allocation | Supporting | DCGM profiling and standard fields | Workload context, not independent health verdicts. Low utilization can mean the job is waiting for data. |
| Temperature, thermal/power violation duration, clocks, throttle reasons, power draw/cap | Supporting; platform-critical alarms are key | DCGM/NVML and BMC | Combine sustained abnormal throttling with comparable-work slowdown. Expected power capping alone is not a fault. Use vendor platform safety alarms directly. |
| PCIe negotiated width/speed under load, replay/error rates, AER events; NVLink error/recovery rates | Supporting; fatal/unrecoverable events are key | DCGM/NVML, kernel AER, platform telemetry | Correlate with transfer/collective regression. Interpret error semantics by generation; link-idle power saving is not a regression. |

Representative DCGM fields include `DCGM_FI_DEV_GPU_TEMP`, `DCGM_FI_DEV_POWER_USAGE`, `DCGM_FI_DEV_SM_CLOCK`, `DCGM_FI_DEV_GPU_UTIL`, `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL`, `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL`, and `DCGM_FI_DEV_PCIE_REPLAY_COUNTER`. These are examples, not a universal exporter configuration. Field support, units, and exporter names vary with GPU generation and DCGM version [1][2].

Keep the last Xid value distinct from an event count. Prefer a driver event stream plus durable event identifiers; repeated identical Xids can be missed if only a last-value gauge is sampled. Profiling fields can have collection/multiplexing constraints; budget and validate their overhead.

### 4.2 Networking: GPU-to-GPU across nodes

| Signal | Role | Source examples | How to use it |
|---|---|---|---|
| Inter-node collective success, timeout rate, algorithmic/bus bandwidth, tail duration | Key | NCCL tests, distributed job telemetry | Gate on fixed group size, message size, algorithm/protocol, NIC rails, placement, and expected transport. A one-node test is insufficient. |
| RDMA connection/data-transfer success and latency/throughput | Key | Bounded RDMA probe and transport instrumentation | Confirm that the intended RDMA/GPU-direct path still works, not merely that TCP ping succeeds. |
| Required NIC/rail availability, link speed/width and reachability | Key | NIC telemetry, RDMA tools, fabric controller | A missing mandatory rail or failed required connection is a failure after the recovery deadline. |
| Link integrity errors, uncorrectable FEC, link flaps, transport retries/timeouts | Supporting; loss of required service is key | NIC and switch counters | Require new rates and correlate endpoints with switch ports; localize bad cable/port versus host change. |
| InfiniBand transmit wait, RoCE ECN/CNP, PFC pause duration, queue occupancy/drops | Supporting | Fabric controller, switches, NIC counters | Congestion is evidence, not proof of a defective GPU. Normalize by link rate and interval; correlate with collective degradation [6]. |
| Per-rail traffic distribution, fallback transport, CPU/NUMA locality | Supporting | NCCL logs/telemetry, NIC counters, placement data | Detect one unused rail or silent RDMA-to-TCP fallback. A fallback is a capability failure if RDMA is required by contract. |

Report both end-to-end collective impact and the affected fabric scope. Two endpoints reporting the same bad link are one fault, not two independent node failures.

### 4.3 Storage and the GPU data path

| Signal | Role | Source examples | How to use it |
|---|---|---|---|
| Read/write correctness, uncorrectable media/I/O errors, filesystem availability | Key | NVMe/OS/storage-client telemetry, safe checksum probe | New integrity failure or loss of a required data/checkpoint path is a hard gate. |
| Matched read throughput, p95/p99 latency, timeout/error rate | Key | Storage client and controlled read probe | Fix dataset, I/O size, queue depth, cache state, locality, and concurrency. Measure the path used by jobs. |
| Checkpoint completion, duration, and verified restore on a test artifact | Key | Framework/checkpoint telemetry | Needed for updates that affect storage or recovery; write only to dedicated test locations with quotas. |
| Expected GPUDirect Storage path active and functional | Key when required; otherwise supporting | GDS support checks, cuFile/nvidia-fs telemetry, targeted GDS I/O | Detect unsupported or failed direct I/O and unexpected fallback. A capability check alone does not establish throughput [7]. |
| GPU input-wait fraction and data-loader wait | Supporting | Framework/job instrumentation | Combine with lower storage throughput or higher latency. Not a universally available DCGM metric. |
| Disk utilization/queue depth, CPU I/O wait, NVMe health/temperature, storage network congestion, cache hit rate | Supporting | OS, NVMe, storage backend, NIC/switch telemetry | Distinguish local driver regression, shared storage incident, and workload/cache effects. |

### 4.4 Host and management signals

Always collect host CPU/memory pressure, OOM events, driver/module/service state, kernel faults, scheduler readiness, collector health, time synchronization, BMC power/cooling alarms, PSU/fan status, and hardware event logs. These are supporting context except for required-service loss, host failure, and platform-critical alarms.

The BMC may expose only a subset of GPU sensors and cannot generally validate CUDA, NCCL, GPU memory correctness, or the container data path. Redfish/events and management polling are platform-dependent and may be slower than host telemetry.

## 5. Comparing before and after without false conclusions

### 5.1 Establish an eligible baseline

- Capture at least 30 minutes of recent healthy, representative operation before the update; retain 24 hours for workload variation and rare-event context.
- For a drained node, also run the same bounded probe suite immediately before and after the change. Production history alone may not contain comparable work.
- Exclude already-unhealthy nodes from canary eligibility. Keep their history for incident analysis; do not normalize an existing critical fault into "acceptable."
- Group baselines by GPU SKU/count/MIG profile, NIC/fabric topology, storage tier/path, power policy, workload shape, software manifest, and test version.
- Keep unchanged control nodes in the same relevant failure domain, plus reference controls outside that domain where possible. Label shared fabric and storage dependencies.
- If no comparable baseline/control exists, return `HOLD` pending controlled validation or explicit, audited operator acceptance.

### 5.2 Calculate comparable regressions

For throughput, higher is better:

```text
local_drop = 1 - post_updated / pre_updated
control_adjusted_drop =
    1 - (post_updated / pre_updated) / (post_control / pre_control)
```

For latency, lower is better:

```text
local_rise = post_updated / pre_updated - 1
control_adjusted_rise =
    (post_updated / pre_updated) / (post_control / pre_control) - 1
```

Require positive, valid denominators, aligned measurement windows, and comparable workloads. These relative formulas are not valid for zero-baseline error counters. For errors, use new event counts, exposure-normalized rates, and absolute vendor/platform limits.

Use the median across valid throughput repetitions; compute tail latency from raw observations or compatible merged histograms, not an average of p99 values. Require a regression to exceed both the configured minimum effect and the measured baseline noise envelope. Start with a noise envelope of three robust standard-deviation equivalents derived from baseline MAD; if MAD is zero, retain the minimum-effect threshold.

Controls help attribution, but **do not excuse an absolute availability, correctness, or workload-SLO breach**. If both updated and control nodes degrade, hold the rollout as a shared incident rather than declaring the update safe.

### 5.3 Handle counters and resets correctly

- Use deltas/rates, not lifetime totals, for ECC, replay, network, and I/O counters.
- Track boot ID, device reset epoch, collector epoch, and counter width/reset semantics. A reset begins a new series; never turn a negative delta into a healthy zero.
- Unsupported, blank/sentinel, stale, permission-denied, and collector-error values are distinct from numeric zero.
- Learn a new performance baseline only after release acceptance. Never let a rolling baseline absorb the candidate regression while deciding promotion.
- Preserve raw event timestamps across reboot; distinguish delayed historical events from new post-update faults.

## 6. Decision logic: which signals stand alone?

### 6.1 Hard gates: one confirmed condition is enough

Any of the following produces `FAIL` and `HALT` for the active rollout wave, with containment scoped to the affected resource:

1. New uncorrectable GPU memory error, confirmed computation/data-integrity failure, or row-remapping failure.
2. A vendor-classified fatal/unrecoverable GPU/NVSwitch fault, GPU lost from the bus, or recovery-required state that makes a required resource unsafe/unavailable.
3. Missing/unresponsive required GPU, required NVLink/NVSwitch fabric, NIC rail, storage path, or runtime GPU access after its recovery deadline.
4. Confirmed collective/probe failure on a reserved canary after one bounded infrastructure-aware retry, or a required production communication path demonstrably unavailable. A verified real correctness error does not wait for a retry.
5. Platform-critical power/thermal/cooling alarm according to OEM limits.
6. Host fails to return and pass required readiness checks before its update-specific deadline.

Confirmation means valid identity, current event/state, appropriate severity classification, and trustworthy source. It does **not** mean waiting for a second metric to confirm a genuine uncorrectable error.

A workload-specific illegal-access Xid, application OOM, historical ECC count, or ordinary power-cap throttle is not automatically a hard gate. Classify and correlate it. Keep the Xid policy keyed to driver/GPU generation and vendor guidance [4].

### 6.2 Soft gates: combine impact with evidence

Evaluate each rule in one-minute windows. Initially require **three consecutive eligible bad windows**, with the supporting condition present in each. Soft `HALT` rules require a comparable-work match, adequate coverage, and an effect beyond baseline noise. If control evidence is absent or conflicting, use `HOLD` unless an absolute gate fails.

| Rule | Primary impact X | Corroboration Y | Decision |
|---|---|---|---|
| Compute regression | Control-adjusted compute throughput drop >10% | New sustained thermal/hardware-slowdown activity, clocks >10% below matched expectation, or classified GPU/memory warning | `HALT` |
| GPU-local fabric regression | Intra-node collective bandwidth drop >15% or p99 duration rise >20% | New abnormal NVLink/PCIe error rate, degraded required topology, or fabric health warning | `HALT`; missing required topology already uses a hard gate |
| Cluster fabric regression | Inter-node collective bandwidth drop >15% or p99 duration rise >20% | New transport retries/timeouts, abnormal congestion/pause duration, rail imbalance, or unintended transport change | `HALT` when canary-local; `HOLD` if controls show a shared incident |
| Storage/data-feed regression | Comparable storage throughput drop >20% or p99 latency rise >25% | Increased GPU input wait, new I/O timeout rate, unintended GDS fallback, or a storage-client health warning | `HALT` when candidate-local; otherwise `HOLD` |
| Memory deterioration | Correctable ECC rate above the SKU-specific warning threshold and healthy baseline | Compute/HBM regression or a vendor-classified memory-health warning | `HALT`; isolated correctable errors alone produce `WARN` |
| Host-agent interference | Job throughput drop >10% or p99 step latency rise >20% | Agent CPU/memory overhead or NUMA/CPU contention exceeds its measured budget | `HALT` |

For clock/throttle and congestion conditions, require activity above a calibrated platform-specific threshold, not merely a nonzero counter. Do not set one temperature, ECC-rate, PCIe-replay, or PFC threshold for every GPU and fabric generation.

**Important exceptions to "X AND Y":**

- An end-to-end compute or collective throughput drop >20%, or p99 duration rise >30%, can halt **without an explanatory counter** if reproduced in two independent reserved-canary runs against stable controls. Each run must exceed the noise envelope and its own minimum sample requirements. Silent regressions must not escape because hardware counters look clean.
- One strong primary regression without corroboration produces `HOLD` and an immediate targeted retest, not `CONTINUE`.
- A required production SLO breach can halt independently under that workload's approved policy.
- GPU utilization dropping by itself, temperature increasing within normal limits, or a lifetime error count remaining nonzero does not stop deployment.

Do not use arbitrary "two bad metrics" logic: temperature and thermal throttling can be two views of the same cause. Require an impact measure plus a relevant explanatory signal, or repeated independent end-to-end evidence.

### 6.3 Freshness and evidence gates

Initial collection and decision targets:

- Host heartbeat every 5 seconds; heartbeat older than 15 seconds makes host observability `UNKNOWN`.
- Required fast metrics every 5 seconds; older than 20 seconds means stale.
- Critical event delivery: target p99 <=10 seconds from source emission to orchestrator fence acknowledgement.
- Polled hard-state detection: target p99 <=20 seconds including sampling.
- Soft regression verdict: target <=5 minutes after sustained degradation begins, once sufficient eligible workload/probe samples exist.
- Minimum eligible coverage: 95% of expected fast samples per evaluated entity/window, no stale required source, and at least 30 matched job iterations or independent probe observations per window. Require more for statistically useful p99 estimation; 30 is not enough to claim reliable p99.

These are engineering SLOs to measure, not guarantees. A BMC sensor refreshing every minute cannot meet a ten-second source-detection target. Declare slower sources and update-phase expectations explicitly.

Planned collector/host restarts have bounded, per-phase maintenance allowances. During an allowed gap, promotion remains blocked until recovery and evidence collection complete. Outside that allowance, stale data immediately holds the rollout; a missed recovery deadline halts it. Missing optional sensors produce a visible coverage warning, not a fabricated healthy sample.

## 7. Rollout state machine and enforcement

```text
BASELINE
   -> PRECHECK
   -> DRAIN / CHECKPOINT
   -> APPLY_UPDATE
   -> RECOVER
   -> VALIDATE
   -> SOAK
   -> PROMOTE_NEXT_WAVE

Any hard gate or confirmed regression -> HALTED
Missing/ambiguous evidence           -> HELD
HALTED -> TRIAGE -> approved ROLLBACK or REPAIR -> REVALIDATE
HELD   -> evidence/retest            -> VALIDATE or HALTED
```

1. **Baseline/precheck:** verify all mandatory sources, topology, test capacity, rollback prerequisites, and unchanged controls. Persist the policy and baseline used for the decision.
2. **Drain/checkpoint:** coordinate with the scheduler. For distributed jobs, checkpoint/restart the job group as required; a single rank cannot always be safely drained independently.
3. **Apply/recover:** declare expected disruptions and per-component recovery deadlines. Planned link/service loss is not a surprise fault, but it never permits promotion. Integrity and platform safety alarms are not suppressed.
4. **Validate:** check inventory, service/runtime access, memory state, and the update-specific probes. Run intrusive GPU/DCGM diagnostics only on reserved/drained resources, never opportunistically on customer jobs [5].
5. **Soak:** start the performance window only after initialization, warm-up, and comparable work are available. Initial proposal: 15-30 minutes for low-risk host agents; at least 60 minutes and cross-node probes for driver, fabric, firmware, or storage changes. Rare failures may require longer exposure.
6. **Promote:** require explicit fresh `CONTINUE`, complete coverage, and all required node/group gates. A canary with no eligible work stays held.
7. **Halt:** stop dispatching updates, cordon/quarantine the affected resources where appropriate, preserve evidence, and notify owners. Do not abruptly terminate unrelated running jobs solely to express a rollout halt.

Suggested rings: lab -> one complete representative communication group -> a small topology-diverse cohort -> larger waves -> fleet. "One GPU server" is not a sufficient first canary for a multi-node communication change. Keep control capacity unmodified until the candidate is accepted.

**Blast-radius policy:**

- One hard fault in an active canary immediately fences the active deployment wave; do not wait for a fleet failure percentage.
- A local resource failure quarantines that resource and any inseparable job/fabric group, not every server in the cluster.
- The same candidate-associated failure signature on two independent nodes/groups in separate relevant failure domains halts that release across its deployment scope. Reports of the same shared switch fault do not count as independent failures.
- A shared storage/fabric incident holds affected waves even when update attribution is unknown.
- Ordinary faults outside the candidate cohort still follow incident handling; they are not automatically attributed to this release.

The orchestrator uses short-lived promotion permits tied to `deployment_id`, wave, manifest, policy version, and topology scope. No fresh permit means no new update. Revoke permits on `HOLD`/`HALT`; check them at each dispatch, limit in-flight updates, and fence stale workers. Already-applied changes cannot be magically cancelled.

Make halt commands idempotent and durable. Require orchestrator acknowledgement, retry until acknowledged, and alert if acknowledgement misses its deadline. Record in-flight work at the fence. Resuming a halted release requires an audited decision and successful revalidation; a few good samples must not automatically resume it.

Roll back only through a supported component-specific plan. Driver/library/firmware compatibility, firmware downgrade restrictions, and distributed-job state can make blind rollback unsafe. Firmware failures may require isolation and repair instead.

## 8. Why GPU updates differ from normal cluster updates

| Conventional cluster gate | Additional GPU-cluster requirement |
|---|---|
| Node ready, service probes, CPU/memory, request errors/latency | Verify every expected GPU, CUDA/runtime access, memory correctness, fabric readiness, and delivered GPU performance |
| Nodes treated mostly as independent replicas | Nodes participate in tightly coupled collective jobs; the slowest rank or broken rail can stall the entire job |
| Network reachability/HTTP health is adequate for many workloads | Test actual RDMA/GPU-direct transports, all required rails, and representative collectives |
| Node is the main maintenance unit | Maintenance may span a job group, NVLink domain, fabric partition, rack, or shared management service |
| Image and service compatibility | Validate kernel/driver/GSP/Fabric Manager/NVLSM/CUDA/NCCL/NIC/GDS compatibility and MIG configuration |
| Rolling replacement with spare replicas | Check expensive GPU spare capacity, checkpoint duration, rank placement, and restart feasibility |
| Generic utilization and latency averages | Match workload shapes, precision, topology, power caps, and cache state; inspect worst GPU/rank/rail |
| Service restart usually restores the stack | GPU reset, fabric reinitialization, reboot, or power cycle may be required and have different scopes |
| Brief request-based smoke tests | Include compute/HBM, intra-node, inter-node, data-feed/checkpoint, and correctness tests as relevant |

Most importantly, a driver can report healthy and a Kubernetes node can report `Ready` while distributed training is substantially slower. GPU health must include **useful work and communication**, not only process liveness.

## 9. Architecture and near-real-time signal flow

```text
 GPU / driver / runtime       NIC / RDMA       Storage / workload
          |                       |                   |
          +----------- host collectors --------------+
                                  |
                      local observation adapter
                                  |
                    HSDU host client / local gate
                     | durable spool | heartbeat
                     |               |
                     +---- authenticated push -----------+
                                                         |
 BMC -- management collector -- independent push -------->|
 SoC/DPU collector -- independent push (if present) ----->|
 Fabric/switch controller -- topology-scoped push ------->|
                                                         v
                                               ingestion gateway
                                                         |
                                             durable partitioned stream
                                                 /               \
                                      HealthStore history     online evaluator
                                      and baseline queries       |
                                                                 v
 Deployment events + manifests ------------------------> rollout gate
                                                                 |
                                               durable HOLD/HALT/CONTINUE
                                                                 |
                                                  deployment orchestrator
                                                    fence + acknowledge
```

### Responsibilities

- **Collectors/adapters:** use DCGM/NVML and driver events for GPUs; supported NIC/RDMA, OS, storage, and job instrumentation for other domains. Capture actual observations, not just an agent-authored "healthy" string.
- **HSDU on host:** normalize identity/units, validate schema, attach deployment context, detect local hard faults, spool durably, and push promptly. A local hard-fault gate can request immediate containment; absence of local failures cannot authorize fleet promotion.
- **Independent sources:** observe host loss, power/cooling, switch health, and optional DPU state even if the host observer fails.
- **Ingestion/stream:** authenticate producers, validate schema, persist before acknowledging, partition for per-entity ordering, and preserve a separate priority path for critical events.
- **Online evaluator:** maintain event-time windows, freshness, counter epochs, baselines, controls, topology correlation, and policy decisions. Join by deployment and affected dependency graph.
- **HealthStore:** retain raw observations, manifests, test artifacts, decisions, and acknowledgement history. Do not put a slow batch warehouse query on the critical-event path.
- **Rollout gate/orchestrator:** persist fences and require fresh positive authorization for promotion.

If existing HealthStore already supports low-latency durable ingestion, stream evaluation, and command delivery, reuse those capabilities. Otherwise add an adapter/stream evaluator beside it. This design does not assume those interfaces already exist.

### Delivery semantics

- Push over authenticated gRPC/HTTP streaming or an existing supported event bus; use mTLS/workload identity and authorize node identity at ingestion.
- Use at-least-once delivery with acknowledgement **after durable acceptance**, bounded retry/backoff, and event-ID deduplication. Do not claim exactly-once transport.
- Keep per-producer sequence numbers and separate producer/boot epochs. Device counter epochs are tracked independently.
- Carry source time, collection time, and server receipt time. Use monotonic time for local durations and heartbeat expiry measured at the receiver; detect clock skew rather than silently misordering evidence.
- Late hard-fault events still trigger incident evaluation. Bounded late data can recompute soft windows, but replayed history must never turn into a fresh promotion permit.
- Reserve spool/stream capacity for hard events and heartbeats. When overloaded, downsample noncritical metrics explicitly and emit a data-quality event. Lost required coverage holds the rollout.
- Expose queue depth, oldest-event age, dropped/rejected observations, per-source freshness, and gate-ack latency.
- Separate the collector/client rollout from the candidate agent where possible. If updating HSDU/DCGM itself, keep independent observer coverage and validate schema/metric parity before promotion.

An example ten-second critical-event budget is: <=1 second local capture, <=2 seconds publish/durable acceptance, <=2 seconds evaluation, and <=5 seconds fence delivery/acknowledgement. Validate the end-to-end p99 directly; component p99 values do not mathematically guarantee the combined p99.

## 10. Integrating the existing CSV-based HSDU flow

### Preferred path

Add a local structured observation API, such as a Unix-domain socket or supported local RPC endpoint. HSDU receives events immediately, writes its spool, and pushes them off-node. Use periodic snapshots for reconciliation, not as the only critical-event channel.

For existing producers, add a **CSV compatibility adapter**. This allows incremental migration without requiring all agents to change immediately.

### Safe transitional CSV contract

If CSV remains the initial interface:

1. Use a separate snapshot per producer; never allow unrelated agents to append concurrently to one shared file.
2. Write a complete temporary snapshot and atomically rename it on the same filesystem. Test the chosen filesystem's rename/durability behavior.
3. Include schema version, producer ID, producer epoch, sequence number, observed timestamp, entity identity, signal name, value/unit, and quality. Include boot/device epochs where applicable.
4. Watch file-change notifications, with a one-second reconciliation scan to catch missed notifications. File modification time alone does not establish signal freshness.
5. Reject malformed, truncated, out-of-order, or unsupported records explicitly. Emit a parse/data-quality event and keep the last valid value marked stale when it expires; do not silently substitute `healthy`.
6. Track producer freshness, not just file existence. A dead agent's old "healthy" row must time out.
7. Use a separate durable event queue/API for transient hard events. A snapshot can overwrite an error before HSDU reads it. If files are unavoidable, use a validated append-only event journal with sequence numbers, rotation/cursor recovery, and durable acknowledgements.
8. HSDU's off-node spool retries until accepted and survives its own restart. Disk-full or spool-corruption conditions emit an independent alarm and hold promotion.

Suggested transitional columns:

```text
schema_version,producer_id,producer_epoch,sequence,node_id,boot_id,entity_id,device_epoch,signal,observed_at,value,unit,quality,deployment_id
```

Represent quality explicitly, for example `VALID`, `UNSUPPORTED`, `COLLECTOR_ERROR`, or `INVALID`; derive `STALE` at evaluation time. Do not encode missing/unsupported as zero.

If an agent can only publish a binary CSV status, retain it as an advisory signal, but acquire critical GPU and fabric evidence independently. An updated agent cannot be the sole witness to its own correctness.

### Example normalized event

This is a proposed envelope, not an existing HSDU API. The sample identifier and timestamp are illustrative.

```json
{
  "schema_version": 1,
  "event_id": "node-017:boot-a:dcgm-epoch-3:9241",
  "kind": "metric.observation",
  "node_id": "node-017",
  "boot_id": "boot-a",
  "producer": {
    "id": "gpu-collector",
    "version": "1.0.0",
    "epoch": "dcgm-epoch-3",
    "sequence": 9241
  },
  "observed_at": "2026-11-15T07:00:00Z",
  "collected_at": "2026-11-15T07:00:00Z",
  "deployment": {
    "id": "deploy-042",
    "wave": "canary",
    "phase": "SOAK",
    "component": "host-management-agent",
    "from_version": "4.2.0",
    "to_version": "4.3.0"
  },
  "entity": {
    "type": "gpu",
    "id": "GPU-example-uuid",
    "counter_epoch": "gpu-reset-2"
  },
  "topology": {
    "rack_id": "rack-03",
    "fabric_partition": "partition-a"
  },
  "signal": {
    "name": "gpu.memory.uncorrectable_errors",
    "type": "counter",
    "value": 1,
    "unit": "errors",
    "quality": "VALID"
  }
}
```

The evaluator derives a **new error** only if it has a valid preceding sample in the same counter epoch, or an authoritative new-error event. A first-seen cumulative value of `1` does not prove a new post-update error. Missing prior evidence holds the relevant gate until reconciled.

An emitted decision should contain at least:

```json
{
  "decision_id": "decision-812",
  "deployment_id": "deploy-042",
  "wave": "canary",
  "action": "HALT",
  "scope": {
    "type": "deployment_wave",
    "id": "canary"
  },
  "affected_entities": ["node-017"],
  "reason_code": "GPU_NEW_UNCORRECTABLE_MEMORY_ERROR",
  "policy_version": "gpu-gate-v1",
  "baseline_id": "baseline-042",
  "evidence_ids": ["observation-before", "observation-after"],
  "attribution": "UNDETERMINED",
  "requires_acknowledgement": true
}
```

Persist both the decision and the orchestrator acknowledgement. Store explicit eligibility/expiry on `CONTINUE` permits; `HALT` remains latched until audited resolution.

## 11. Should HSDU move to the SoC?

**Not as a prerequisite, and not as a wholesale replacement for the host client. Use a split design.**

| Placement | Best visibility | Important limitations | Recommendation |
|---|---|---|---|
| Bare-metal host | GPU APIs/events, CUDA, runtime access, application performance, OS/storage stack | Can disappear during host failure/update; shares resources with workloads | Primary HSDU collector/client |
| SoC/DPU, if present | Independent heartbeat, DPU/NIC/offload path, some management observations | GPU telemetry/access is platform-specific; cannot assume host CUDA/NCCL visibility; may share power/network failures | Optional independent collector, relay, and watchdog |
| BMC/out-of-band collector | Power/cooling, hardware logs, host power state | Limited sensor refresh and application visibility; affected by BMC updates | Independent hardware-health evidence |
| Off-node control plane | Fleet history, controls, correlation, durable deployment fences | Depends on ingestion/connectivity and good identity/topology data | Authoritative rollout evaluation and enforcement integration |

Moving everything to a DPU would not solve host-library regressions, job correctness, or data-loader stalls. A relay on the DPU can improve survivability if the platform supports it, but a separate management path and failure-domain analysis matter more than the processor label.

If HSDU itself is required to remain alive through host reboot, consider an SoC-resident **supervisor/relay** while retaining a host helper for in-band evidence. Confirm supported access, signing/upgrade ownership, resource isolation, and management-network reachability first. During DPU or BMC updates, do not treat that same component as the sole independent observer.

## 12. Update-specific validation profiles

All profiles retain common correctness, availability, and freshness gates; profiles add tests rather than disabling unrelated hard faults.

| Candidate update | Additional mandatory evidence |
|---|---|
| Host/security/management agent | Agent overhead budget, GPU device permissions, runtime probe, comparable job throughput, basic collective/data-feed check |
| HSDU/DCGM/telemetry configuration | Independent observation, old/new field and unit parity, event loss/duplication/restart tests, gate delivery and coverage |
| Driver/kernel/GPU firmware | Compatibility manifest, all-GPU compute/memory correctness, GPU topology, intra/inter-node communication, reset/reboot recovery |
| Fabric Manager/NVLSM/NVSwitch firmware | Fabric initialization and partitions, all required links, intra-node collectives, representative inter-node group, recovery scope |
| NIC/RDMA/DPU/network agent | Every required rail, GPU-direct transport where required, RDMA latency/throughput, collectives, endpoint/switch correlation |
| Storage client/GDS/NVMe firmware | Correct I/O path, read correctness/performance, safe checkpoint/restore, data-feed wait and fallback detection |
| BMC/BIOS/CPLD/retimer firmware | Independent management observer, post-reboot sensors/power policy, PCIe topology/training under load, inventory and GPU smoke suite |
| CUDA/NCCL/framework image | Runtime/ABI checks, numerical correctness, representative workload shapes and collective algorithms/message sizes |

Tests need explicit duration, timeout, resource ownership, retry, and cleanup budgets. Do not run full-memory diagnostics, power stress, large distributed tests, or storage writes on production workloads without scheduler reservation and an approved profile.

## 13. Implementation sequence

1. **Inventory and policy contract:** confirm HSDU interfaces, SoC meaning/capabilities, hardware generations, fabric/storage stack, deployment events, scheduler integration, and orchestrator fencing API.
2. **Minimum viable safety gate:** host heartbeat, GPU inventory/runtime access, classified fatal events, new uncorrectable memory errors, required fabric readiness, and durable halt/acknowledgement. Start with CSV compatibility where needed.
3. **Performance gate:** capture fixed baselines and unchanged controls; add compute/HBM, intra-node, inter-node, and storage/data-feed probes.
4. **Shadow evaluation:** record decisions without halting; replay historical incidents and quantify false positives, missed failures, detection delay, and source overhead.
5. **Canary enforcement:** enable blocking first for hard gates, then calibrated soft rules. Require explicit human acceptance before expanding enforcement.
6. **Independent telemetry and resilience:** integrate BMC/fabric/DPU sources as available and validate host/observer/network failure behavior.

### Acceptance scenarios before production

| Scenario | Expected result |
|---|---|
| New verified uncorrectable GPU error during canary | Immediate hard gate; fence acknowledged within the measured critical-event target |
| GPU utilization falls because there is no queued work | No false hardware failure; performance remains unvalidated until eligible work exists |
| 12% matched compute regression with sustained abnormal throttling for three eligible windows | Soft halt if candidate-local and beyond the baseline noise envelope |
| 25% reproducible collective throughput loss with clean error counters | Halt after two independent controlled runs against stable controls |
| Shared switch/storage problem affects controls and candidate nodes | Hold affected rollout; create shared-infrastructure incident rather than claim update causality |
| Host/collector reboots and ECC counter restarts | No negative-rate artifact; epoch transition tracked and missing evidence explicitly gated |
| Last-value Xid gauge repeats the same code | Event stream preserves separate occurrences; no assumption that code arithmetic is meaningful |
| Agent keeps an old healthy CSV on disk or sends malformed rows | Required source becomes unknown; rollout cannot promote |
| Collector updates change metric units or remove a required field | Explicit schema/capability failure; hold rather than compare incompatible values |
| Expected restart within declared maintenance window | Promotion blocked; resume evaluation only after readiness and coverage recover |
| Host, DPU, broker, evaluator, or orchestration connection fails | No new promotion permit; spooled replay/deduplication and fence recovery work as specified |
| GPU absent after recovery deadline or required transport silently falls back | Hard availability/capability failure under the declared profile |
| Replayed old events, delayed samples, duplicate halt commands, or stale permits | No false promotion; idempotent fencing and preserved audit history |

Measure source-to-fence latency, sustained-regression-to-decision latency, false-halt rate per update wave, missed incidents, required-source coverage, observer overhead, and acknowledgement reliability. Exercise actual failure injection in a lab; a dashboard alone does not validate the safety mechanism.

## 14. Decisions to confirm before coding

- What HSDU stands for internally, which APIs it already exposes, and whether HealthStore supports durable low-latency streams and online evaluation.
- Whether SoC means DPU/management SoC here and which GPU/host observations it can actually access.
- GPU models/generations, expected GPU/MIG/link/rail topology, and supported driver/firmware combinations.
- Kubernetes, Slurm, or another scheduler; required checkpoint/drain behavior and available spare capacity.
- InfiniBand versus RoCE/Ethernet, fabric-controller ownership, and switch telemetry access.
- Local versus remote storage, required GDS behavior, and application performance/SLO instrumentation.
- Deployment orchestrator pause/fence APIs, in-flight update limits, rollback ownership, and release-resume authority.
- Approved per-profile thresholds, sample sizes, recovery deadlines, soak durations, and independently measured delivery SLOs.

## 15. References and evidence boundaries

The architectural choices and threshold values are recommendations in this README. Vendor documentation supports the capabilities and cautions below; it does not prescribe this rollout policy.

1. [NVIDIA DCGM health monitoring](https://docs.nvidia.com/datacenter/dcgm/latest/learn/modules/health-monitoring.html): passive subsystem monitoring, including memory, PCIe, thermal/power, and GPU fabric.
2. [NVIDIA DCGM field identifiers](https://docs.nvidia.com/datacenter/dcgm/latest/reference/field-identifiers.html) and [DCGM Exporter metrics](https://docs.nvidia.com/datacenter/dcgm/latest/reference/dcgm-exporter-metrics.html): field definitions, units, exporter semantics, and version-dependent support.
3. [NVIDIA GPU Operator overview](https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/overview.html) and [Fabric Manager guide](https://docs.nvidia.com/hgx-platforms/fabric-manager-user-guide/index.html): GPU software management and platform-specific fabric services, including NVLSM on applicable systems.
4. [NVIDIA Xid catalog analysis](https://docs.nvidia.com/deploy/xid-errors/latest/analyzing-xid-catalog.html): error interpretation and recovery require context rather than treating every code identically.
5. [NVIDIA DCGM diagnostics](https://docs.nvidia.com/datacenter/dcgm/latest/learn/modules/dcgm-diagnostics.html): active diagnostics exercise resources and require coordination; they do not prove a failed component is eligible for RMA.
6. [NVIDIA UFM telemetry fields](https://docs.nvidia.com/networking/display/ufmenterpriseumv6222/high-frequency+(primary)+telemetry+fields): fabric counter availability and collection characteristics.
7. [NVIDIA GPUDirect Storage troubleshooting guide](https://docs.nvidia.com/gpudirect-storage/troubleshooting-guide/contents.html): path support, diagnostics, and storage-specific statistics.

Exact Azure-internal agent composition, private HSDU contracts, and SoC placement constraints are not established by these public references. Validate them with the platform owners before selecting adapters or approving production enforcement.
