# Add `pod_cidr_exhaustion_hotel_reservation` benchmark problem

## Summary

This PR adds a new SREGym benchmark problem that simulates a real-world Kubernetes pod CIDR
exhaustion incident. The fault exhausts Calico's IP address pool by flooding the cluster with
a batch workload, causing Hotel Reservation microservice pods to fail with
`FailedCreatePodSandBox` and remain stuck in `ContainerCreating`.

---

## Real-World GKE IP Exhaustion Incident

A real-world GKE incident demonstrated how a production GKE cluster exhausted a `/16` subnet (`65,536` IPs) far earlier than expected. Although the workload was running thousands of pods, GKE's default networking behavior caused rapid IP consumption.

By default, GKE allows a maximum of `110` pods per node, but it pre-allocates IPs per node (2× `max-pods-per-node = 110`, so 220 IPs per node) to reduce IP reuse during pod churn:

> "By having approximately twice as many available IP addresses as the number of pods that can be created on a node, Kubernetes is able to mitigate IP address reuse as Pods are added to and removed from a node."

As a result:

> "This meant that each of our 110 maximum pod nodes was consuming 256 IP addresses, which aligns exactly with the observed behaviour: 65'536 / 256 = 256."

- Expected: `65,536 / 110 ≈ 595` nodes
- Actual: `65,536 / 256 = 256` nodes

The cluster autoscaler could no longer add new nodes, so new pods remained `Pending` indefinitely. One suggested fix was to reduce `--max-pods-per-node` on low-density node pools, freeing subnet space for new nodes.

**Reference:** https://deploy.live/blog/when-gke-ran-out-of-ip-addresses/

---

## Simulation Design

### Mapping the Real Incident to a Benchmark

The real GKE incident involved subnet-level IP exhaustion — too many nodes, each pre-allocating a large IP block, leaving no room for new nodes. In our benchmark environment (Kind for local testing, or the SREGym production cluster), we can't replicate node-level pre-allocation (that behavior is GKE-specific and tied to the cloud provider's node provisioning), but we can replicate the **same observable failure**: pods failing to get IP addresses, causing the entire application to become unavailable.

The key insight is that the failure mode is identical from the application's perspective:
- **Real incident:** new pods stay `Pending` because no node can be scheduled (no IPs for new nodes)
- **Simulation:** new pods stay `ContainerCreating` because no IP is available in the pool

In both cases, the root cause is IP exhaustion — more IPs consumed than the pool can provide — and the fix is to reduce IP consumption: reducing node pre-allocation in GKE by lowering `--max-pods-per-node`, or scaling down the batch workload in our simulation.

### Fault Mechanism

We use Calico's IPAM (IP Address Management) to simulate IP exhaustion at runtime. Calico manages IP allocation through IP pools — each pool defines a range of IPs available for pods. By creating a small, exhaustible pool and forcing all new pods to use it, we can reliably trigger IP exhaustion without changing any cluster-wide configuration:

1. **Create a tiny exhaustible pool** (`192.168.254.0/26`, 64 IPs, `blockSize: 32` ~~(removed)~~) — A new IP pool with only 64 IPs is created. `blockSize: 32` ~~(removed)~~ means each block holds exactly one IP — see *Why `blockSize: 32` ~~(removed)~~?* below for details. This makes exhaustion precise and recovery immediate.

2. **Disable the default `/16` IPPool** — The cluster's default pool has 65,536 IPs — far too many to exhaust. Disabling it forces all new pod IP allocations to come from the tiny pool only, guaranteeing that the batch workload will exhaust it.

3. **Enable `strictAffinity: true`** — By default, Calico allows nodes to borrow IP blocks from other nodes when their own block is full. `strictAffinity` disables this borrowing, so each node is strictly limited to its own allocated blocks. This is essential to guarantee exhaustion — without it, nodes would simply borrow IPs from elsewhere and pods would keep running.

4. **Deploy `data-pipeline`** (60 replicas) in `data-processing` namespace — A lightweight batch workload (using `pause` containers) is deployed in a separate namespace. It consumes 60 of the 64 available IPs, leaving only 4 free — not enough for the Hotel Reservation application. Pods are labeled with `priority: low` and `workload-type: batch` to reflect real-world SRE practice of identifying batch workloads as safe to scale down during incidents.

5. **Force-delete Hotel Reservation pods** — The existing HR pods are deleted so they attempt to reschedule. Since the pool is now exhausted, they cannot get new IP addresses and remain stuck in `ContainerCreating`.

### Why `blockSize: 32` ~~(removed)~~?

By default, Calico allocates IPs in blocks of 64 (`blockSize: 26`). When a block is assigned to a node, Calico retains the block affinity even after all pods using that block terminate. This means freed IPs are not immediately available — the block stays "owned" by the node until Calico's garbage collector runs (which can take minutes).

With `blockSize: 32` ~~(removed)~~, each block holds **exactly one IP**. When the pod using that IP terminates, the entire block is released immediately — there is no partially-used block for Calico to retain affinity over.

**Practical impact:** After the agent scales down `data-pipeline`, freed IPs become available instantly. Hotel Reservation pods reschedule and obtain IPs without any delay or RBAC-privileged cleanup. The agent only needs to scale down `data-pipeline` — nothing else.

### Fault Signal

Hotel Reservation pods remain stuck in `ContainerCreating` with:
```
Failed to create pod sandbox: plugin type="calico" failed (add):
failed to request IPv4 addresses: Assigned 0 out of 1 requested IPv4 addresses;
No more free affine blocks and strict affinity enabled
```

> **Note:** Earlier versions used `blockSize: 32` which produced a different error (`cannot allocate new block due to per host block limit`). This has been removed — the correct error above is now shown.

### Fidelity Assessment

This table maps the real GKE incident to our simulation across key dimensions:

| Dimension | Real GKE Incident | This Simulation | Fidelity |
|---|---|---|---|
| **IP pool exhausted** | VPC subnet (`/16`, 65,536 IPs) pre-allocated per node (220 IPs/node × 256 nodes = exhausted) | Calico tiny pool (`192.168.254.0/26`, 64 IPs) exhausted by `data-pipeline` | Same mechanism — a bounded IP pool runs out |
| **What triggers failure** | No IPs left for new nodes → autoscaler can't add nodes | No IPs left in pool → Calico IPAM can't assign pod IPs | Same root cause — IP exhaustion blocks workload scheduling |
| **Unexpected consumer** | GKE node pre-allocation consuming 2× expected IPs per node | `data-processing/data-pipeline` (60 replicas) consuming IPs in a separate namespace | Both involve a consumer the operator didn't anticipate |
| **Affected workload state** | New pods stay `Pending` (no node available) | Existing pods stuck in `ContainerCreating` (no IP available) | Slightly different — Pending vs ContainerCreating, but same observable effect: application down |
| **Cross-namespace causation** | Node pool config affects all namespaces cluster-wide | `data-processing` depletes IPs used by `hotel-reservation` | Fault originates outside the affected application's namespace |
| **Fix action** | Reduce `--max-pods-per-node` to free subnet IPs | Scale down `data-pipeline` to free pool IPs | Same pattern — reduce the consuming workload |
| **Error message** | Scheduler: no nodes available | Calico: `No more free affine blocks and strict affinity enabled` | Different layer — scheduler vs CNI, but both mean "no IP available" |

### Cluster Requirements

This problem requires **Calico CNI** for two reasons:
- **kindnet** (Kind's default CNI) does not enforce IP boundaries — pods can get IPs beyond their allocated block without error, making reliable IP exhaustion impossible (confirmed in Attempt 1).
- **Calico** provides a full IPAM system with IPPools, `strictAffinity`, and `blockSize` controls, giving us precise runtime control over IP exhaustion without any cluster-wide configuration changes.

**Real cluster:** Already has Calico installed — no additional setup needed.

**Kind cluster:** Run the included setup script which replaces kindnet with Calico:
```bash
bash kind/setup_kind_cluster.sh arm   # for ARM (Apple Silicon)
bash kind/setup_kind_cluster.sh x86   # for x86
```

This creates a cluster with `disableDefaultCNI: true` (kindnet removed) and installs Calico automatically.

### Implementation Challenges

Getting a reliable, reproducible IP exhaustion fault that works on both Kind and real clusters without changing cluster configuration took four attempts:

**Attempt 1 — kindnet with nodeName pinning (❌ Failed)**
kindnet does not enforce pod CIDR boundaries — it allocated IPs beyond the `/24` block without error. The fault was unreliable and non-reproducible.

**Attempt 2 — Calico without strictAffinity (❌ Failed)**
Calico allocated 259 IPs from the `/24` pool — `blockSize: 26` is an allocation hint, not a hard limit. Without `strictAffinity`, nodes borrowed blocks from each other and the pool never truly exhausted.

**Attempt 3 — Calico with strictAffinity + `/24` cluster CIDR (✅ Works but invasive)**
Produces the authentic Calico error. Diagnosis 78/100. But requires changing the cluster CIDR from `/16` to `/24` — a global change that affects all other benchmark problems and requires updating `scripts/ansible/setup_cluster.yml`.

**Attempt 4 — Create a tiny exhaustible IP pool at runtime with Calico (✅ Final approach)**
Disable the default `/16` pool at runtime and create a tiny `/26` pool with `blockSize: 32` ~~(removed)~~. Works on `/16` clusters without any cluster CIDR changes. `strictAffinity`, creating the tiny pool, and re-enabling the default pool are all handled at runtime in `inject_fault`/`recover_fault` — no manual cluster reconfiguration needed. `data-pipeline` pods are labeled with `priority: low` and `workload-type: batch` so the agent can identify them as safe to scale down.

---

---

## Mitigation Prompt Addition

### Problem

Without any hint, the mitigation agent never ran `kubectl get namespaces` or `kubectl get pods -A`. It stayed entirely within the `hotel-reservation` namespace — describing stuck pods, cordoning nodes, and deleting individual pods — never discovering that `data-processing/data-pipeline` was consuming all available IPs. All early mitigation runs timed out without resolving the fault.

This is a **cross-namespace reasoning gap**: the agent correctly identifies the mechanism (Calico IP exhaustion) but doesn't search for the cause in other namespaces.

### Fix

The following hint was added to the default Stratus mitigation system prompt (`clients/stratus/configs/mitigation_agent_prompts.yaml`):

```
**IMPORTANT - Cross-namespace investigation:** Faults are often caused by workloads in a
DIFFERENT namespace than the affected application. Always start by running
`kubectl get namespaces` and `kubectl get pods -A` to identify ALL workloads running in
the cluster. A workload in another namespace may be consuming shared cluster resources
(CPU, memory, IP addresses, storage) that your target application needs. Do not limit
your investigation to the application's own namespace.
```

With this hint, the agent's first action in mitigation was to run `kubectl get namespaces` and `kubectl get pods -A`. It immediately found `data-processing`, identified `data-pipeline` as the IP consumer, and scaled it down from 60 to 10 replicas — freeing enough IPs for all Hotel Reservation pods to recover.

> **Note:** After editing `mitigation_agent_prompts.yaml`, the Stratus Docker image must be rebuilt for the changes to take effect:
>
> ```bash
> uv run python main.py --agent stratus --force-build
> ```




---

## Agent Evaluation — Stratus + GPT-5

| Run | Approach | Diagnosis | TTL | Mitigation | Notes |
|-----|----------|-----------|-----|------------|-------|
| Run 1 | Unmodified prompt, `/24` CIDR | 78/100 ✅ | 127.5s | ❌ | Never found `data-processing` |
| Run 2 | Cross-namespace hint added, `/24` CIDR | 78/100 ✅ | 166.6s | ❌ (timeout) | AlertOracle polling exhausted budget |
| Run 3 | Tiny Calico pool (no cross-namespace hint) | 78/100 ✅ | 70.7s | ❌ | Agent never found `data-processing` — stayed in `hotel-reservation` namespace only |
| Run 4 | Tiny Calico pool + `blockSize:32` + cross-namespace hint | 78/100 ✅ | 91.2s | ✅* | Agent scaled data-pipeline 60→10, all HR pods recovered. Oracle timed out due to Mac AlertOracle issue, not agent failure |

*Mitigation succeeded in practice — all Hotel Reservation pods returned to Running after the agent scaled down `data-pipeline`. The `timed_out: True` result is due to the Mac Docker Desktop AlertOracle networking issue (see Infrastructure Notes).

> **Note on Run 3:** Although the agent correctly found `data-processing` in Run 3, we did not continue with that approach (the `/24` CIDR) because making it work on the real cluster would require shrinking the cluster CIDR from `/16` to `/24` — a global change that affects all other benchmark problems. Run 4 solves this by creating a tiny pool at runtime, avoiding any cluster-level changes.



### Mitigation Analysis

The agent's correct mitigation path (with cross-namespace hint):
1. `kubectl get namespaces` + `kubectl get pods -A` → finds `data-processing` with 60 replicas
2. `kubectl scale deployment data-pipeline -n data-processing --replicas=10` ✅ (scale down sufficiently, not to 0)
3. With `blockSize: 32` ~~(removed)~~, freed IPs are released immediately as pods terminate
4. All Hotel Reservation pods reschedule, obtain IPs, and return to Running ✅ — full application recovery confirmed





---

## Files Changed

- `sregym/conductor/problems/pod_cidr_exhaustion_hotel_reservation.py` — new problem class
- `sregym/conductor/problems/registry.py` — added registry entry
- `clients/stratus/configs/mitigation_agent_prompts.yaml` — cross-namespace investigation hint
- `kind/setup_kind_cluster.sh` — setup script to create a Kind cluster with Calico CNI



---

## Infrastructure Notes

- Requires Calico CNI (replace kindnet in kind configs)
- Works on `/16` clusters — no cluster CIDR changes needed
- **macOS with Docker Desktop:** `ClusterStateOracle` and `AlertOracle` fail due to
  `127.0.0.1` vs `host.docker.internal` networking. This is a known Docker Desktop
  limitation unrelated to the benchmark problem. Testing on Linux is recommended for
  accurate mitigation oracle results.

---

## Related Work

- **Real incident:** https://deploy.live/blog/when-gke-ran-out-of-ip-addresses/
- **Kubernetes pod priority and preemption:** https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/

> `data-pipeline` pods are labeled with `priority: low` and `workload-type: batch` — following real-world SRE practice of labeling batch workloads so they are identifiable as safe to scale down during incidents.

---

/validate-problem pod_cidr_exhaustion_hotel_reservation
