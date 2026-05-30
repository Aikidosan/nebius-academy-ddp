# Debugging Report — Nebius DDP Training Homework

---

## Issues Summary

| # | Issue | Root Cause | Resolution | Debug Time | Execution Time |
|---|-------|-----------|------------|------------|----------------|
| 1 | Wrong CLI flag `--cluster-id` | Docs/flag name mismatch | Use `--parent-id` | ~5 min | seconds |
| 2 | Shared cluster queue | Shared `ddp-run` name | Personal cluster name | ~10 min | seconds |
| 3 | Wrong k8s context name | SkyPilot uses full internal name | Updated to full name | ~10 min | seconds |
| 4 | 403 on shared registry | No access to course registry | Personal registry | ~15 min | ~5 min (console) |
| 5 | Wrong registry URL format | `registry-` prefix not used in URL | Bare ID only | ~5 min | seconds |
| 6 | Image not found | Image never pushed | `docker build` + `docker push` | ~5 min | ~20 min (large base image) |
| 7 | 403 on private registry (k8s pull auth) | No imagePullSecret on nodes | Made registry public | ~30 min | ~2 min (console) |
| 8 | IAM token command unknown | Wrong subcommand | Avoided via public registry | ~5 min | N/A (bypassed) |
| 9 | Truncated logs | Default 1000-line tail | `--tail 0` flag | ~2 min | seconds |
| 10 | Docker build location confusion | WSL vs Windows confusion | Run everything in WSL | ~5 min | N/A (clarification) |
| | | | **Total** | **~92 min** | **~27 min** |

---

## Architecture Overview

Before diving into the debugging chain, it helps to understand how the system layers interact. Every issue below can be traced back to a boundary between two of these layers failing silently or with a misleading error.

```
┌─────────────────────────────────────────────────────┐
│               Developer Workstation (Windows)        │
│  ┌──────────────────────────────────────────────┐   │
│  │                  WSL2 (Ubuntu)               │   │
│  │   nebius CLI ──► sky CLI ──► kubectl         │   │
│  │   docker (via Docker Desktop WSL integration)│   │
│  └──────────────┬───────────────────────────────┘   │
└─────────────────┼───────────────────────────────────┘
                  │ HTTPS
┌─────────────────▼───────────────────────────────────┐
│           SkyPilot API Server (Nebius VM)            │
│   Receives task YAML, resolves k8s context,          │
│   creates pods, streams logs                         │
└─────────────────┬───────────────────────────────────┘
                  │ Kubernetes API
┌─────────────────▼───────────────────────────────────┐
│         mk8s Cluster (ariel-mitiushkin-mk8s)         │
│   ┌─────────────────┐  ┌─────────────────────────┐  │
│   │  Node (rank=0)  │  │     Node (rank=1)        │  │
│   │  H100 GPU       │  │     H100 GPU             │  │
│   │  torchrun       │◄─►    torchrun              │  │
│   │  NCCL           │  │     NCCL                 │  │
│   └────────┬────────┘  └─────────────────────────┘  │
└────────────┼────────────────────────────────────────┘
             │ imagePull
┌────────────▼────────────────────────────────────────┐
│        Nebius Container Registry                     │
│   cr.eu-north1.nebius.cloud/<registry-id>/           │
│   nebius-trainer:v1  (PyTorch + HF stack)            │
└─────────────────────────────────────────────────────┘
```

**Key insight:** Each arrow in this diagram is an authentication boundary and a naming boundary. Most issues in this project occurred at one of these boundaries — wrong name, missing credential, or wrong format.

---

## Debugging Chain of Thought

### Phase 1 — Provisioning (Issues 1–3)

**Mental model at the start:** "Set up cluster → configure kubectl → launch job."

The first issue (`--cluster-id`) exposed a pattern that repeated throughout: **Nebius CLI uses a resource-tree model where every child references its parent via `--parent-id`**, not by a semantic `--cluster-id` flag. This is consistent with Yandex Cloud heritage, but departs from AWS/GCP conventions where flags are resource-type-specific. The fix was trivial but the underlying convention needed to be internalized.

Issue 2 revealed a **shared-state contention problem**. The shared SkyPilot API server with a fixed cluster name `ddp-run` is a single point of contention when multiple students work simultaneously. The insight was to treat the cluster name as a namespace: use a personally namespaced name (`ariel-ddp`) to get an independent execution context on the same shared API server.

Issue 3 was a **naming layer mismatch**. Nebius assigns human-readable display names to clusters (e.g. `ariel-mitiushkin-mk8s`) but SkyPilot registers them under their full internal context name (`nebius-mk8s-ariel-mitiushkin-mk8s-e00qn3h3n6rqax9gr2`). These are two different namespaces for the same resource, and the YAML must use the SkyPilot-side name, not the Nebius console name.

**Phase 1 pattern:** Naming inconsistencies across abstraction layers (Nebius console name ≠ SkyPilot context name ≠ CLI flag convention).

---

### Phase 2 — Container Registry Auth (Issues 4–8)

This was the longest and most expensive phase (~65 min combined). It involved a cascade of four distinct failures that all presented as variants of the same symptom (`ErrImagePull`) but had different root causes.

**Step-by-step reasoning:**

```
403 on shared registry (Issue 4)
        │
        ▼
  "I don't have access → create my own registry"
        │
        ▼
404 not found on new registry (Issue 6)
        │
        ▼
  "Image was never pushed → build and push it"
        │
        ▼
"Entity not found" on push (Issue 5)
        │
        ▼
  "Registry URL format is wrong → drop registry- prefix"
        │
        ▼
403 on personal registry from k8s nodes (Issue 7)
        │
        ▼
  "Nodes have no pull credentials → try imagePullSecret"
        │
        ▼
IAM token command fails (Issue 8)
        │
        ▼
  "Bypass auth entirely → make registry public"
        │
        ▼
✓ Image pulled successfully
```

The critical architectural insight here: **Kubernetes nodes pull images independently from the developer's local Docker daemon.** Local `docker push` succeeds because the developer's machine has credentials (set up via `nebius registry configure-helper`). But the cluster nodes are separate machines with their own identity — they need either an `imagePullSecret` attached to the pod spec, or the registry must be publicly accessible.

SkyPilot adds a further complication: it generates pod specs internally without exposing a native hook to inject `imagePullSecrets` per task. This means the standard Kubernetes imagePullSecret workflow (patch the default ServiceAccount) was unreliable — SkyPilot may create pods in a namespace or with a service account that doesn't inherit the patched default.

The correct production-grade fix would be:
1. Assign a Nebius IAM Service Account to the node group at creation time
2. Grant that service account `container-registry.images.pull` permission on the registry

Making the registry public was the right pragmatic call for a homework environment. In production, public registries are never acceptable for proprietary model images.

---

### Phase 3 — Execution & Log Collection (Issue 9)

Once provisioning was resolved, training ran cleanly across both H100 nodes. The final issue was operational: `sky logs` defaults to a 1000-line tail for performance reasons, which truncated the NCCL initialization section that appeared early in the log. This is a **default-safe behavior** (avoids streaming gigabytes by accident) that becomes a footgun when the requirement is specifically to capture the beginning of the log.

**Fix:** `--tail 0` disables the limit and streams the full log from the start.

---

### Phase 4 — Environment Mismatch (Issue 10)

The cross-OS environment (Windows host + WSL2 guest) creates a split-brain tooling environment. The Nebius CLI, SkyPilot, and Docker are all installed inside WSL, but a developer on Windows instinctively reaches for PowerShell or Windows Terminal. Docker Desktop bridges this via WSL2 integration, but the bridge only works in one direction: WSL can call Docker Desktop's daemon, but Docker Desktop's Windows-side CLI cannot access WSL's `nebius` or `sky` binaries.

**Rule derived:** When working on Windows with cloud-native Linux tooling, commit to WSL as the single terminal environment. Never split commands across shells.

---

## Key Architectural Insights

### 1. Every abstraction layer has its own naming convention
Nebius console names, SkyPilot context names, Kubernetes context names, and CLI resource IDs are four separate namespaces for the same physical resource. Always resolve the name from the layer you're operating in before writing it into a config file.

### 2. Image pull auth is a node identity problem, not a developer identity problem
Local `docker login` / `nebius registry configure-helper` only configures the developer's machine. Kubernetes nodes are independent principals. Auth must be delegated to them explicitly — via imagePullSecret, node service account, or public registry.

### 3. SkyPilot is an abstraction over Kubernetes, not a transparent pass-through
SkyPilot generates pod specs, manages namespaces, and controls service account assignment. Operations that work natively in `kubectl` (e.g. patching the default ServiceAccount) may not propagate into SkyPilot-managed pods. When SkyPilot's abstraction doesn't expose a knob you need, work at the infrastructure level instead (IAM service account on the node group).

### 4. Shared infrastructure requires namespacing
Shared SkyPilot API servers, shared cluster names, and shared registries all become bottlenecks in multi-user environments. Personal naming conventions (`ariel-ddp`, personal registry) are not just cosmetic — they create independent execution contexts that avoid contention and permission conflicts.

### 5. Defaults are safe, not correct
`sky logs` defaults to 1000 lines. `nebius mk8s` uses `--parent-id` not `--cluster-id`. Defaults are chosen for the median use case. When your requirement deviates from the median (capture full log, use non-standard flag), the default becomes a silent footgun.

---

## Future Improvements — Architecture Level

### For the developer workflow

| Problem | Current state | Improved state |
|---------|--------------|----------------|
| Registry auth for k8s nodes | Manual imagePullSecret or public registry | Attach IAM service account to node group at creation time with `container-registry.images.pull` role |
| SkyPilot context name discovery | Manual inspection after cluster creation | Run `sky check kubernetes` immediately after `get-credentials` and copy the context name into YAML |
| Cross-OS tooling split | Commands split between WSL and PowerShell | One-time setup: pin all cloud tooling to WSL, add WSL alias in Windows Terminal profile |
| Log capture | Remember `--tail 0` flag | Wrap in a shell alias: `alias skylogs='sky logs --tail 0'` |
| Shared cluster contention | Runtime discovery of contention | Use `$USER`-prefixed cluster names by convention from day one |

### For the course/platform design

| Problem | Architectural fix |
|---------|------------------|
| Students hit shared registry they can't access | Course YAML template should reference a public base image or document the required IAM group membership upfront |
| SkyPilot context name mismatch | Add a `sky check kubernetes` verification step to Task 3 in the README before asking students to write the YAML |
| imagePullSecret complexity | Document the IAM service account approach as the primary method; imagePullSecret as fallback |
| Docker build takes 20+ min | Pre-push a base image with PyTorch to the shared registry; student Dockerfile extends it with only application-level deps |

### Infrastructure-as-code recommendation

All manual console steps (create registry, set public access, create node group) should be captured in a single provisioning script to make setup reproducible and auditable:

```bash
# Idempotent setup script (pseudocode)
nebius container-registry registry create --name nebius-trainer --public-access enabled
nebius mk8s cluster create --name ariel-mk8s ...
nebius mk8s node-group create --parent-id <cluster-id> --preset gpu-h100-sxm ...
nebius mk8s cluster get-credentials --id <cluster-id> --external
sky check kubernetes
```

Running this once eliminates Issues 1, 3, 4, 5, 7 from the debugging table entirely.

---

## Time Analysis

```
Total project time:  ~119 min
  ├── Debugging:      ~92 min  (77%)
  │     ├── Registry auth cascade:  ~65 min  (71% of debug time)
  │     ├── Naming/context issues:  ~20 min
  │     └── Misc (logs, env):        ~7 min
  └── Execution:      ~27 min  (23%)
        ├── Docker build+push:      ~20 min
        └── Training run (500 steps): ~46 min (billed, not counted above)
```

The registry authentication cascade (Issues 4–8) consumed **56% of total debug time** for a problem that is fully eliminable by attaching an IAM service account to the node group at cluster creation — a 30-second console operation.

---

## Interview Q&A — Principal Solutions Architect, Nebius (UAE/MEA)

*The following questions are grounded in hands-on experience from this project and mapped to the competencies required for the role.*

---

### Technical Depth — AI Infrastructure & Distributed Training

**Q: Walk me through how PyTorch DDP works across multiple nodes. What are the critical infrastructure requirements?**

> PyTorch DDP (Distributed Data Parallel) distributes gradient computation across GPUs using an all-reduce operation — each GPU computes gradients independently, then they are averaged across all ranks before the optimizer step. Across multiple nodes, this requires:
> - A reliable high-bandwidth, low-latency network between nodes (InfiniBand or RDMA preferred; we fell back to Ethernet/NCCL Socket transport in this project)
> - A rendezvous mechanism: `torchrun` uses a master address (`MASTER_ADDR`) and port (`MASTER_PORT`) for the head node to coordinate rank assignment
> - NCCL as the communication backend — it auto-negotiates the best available transport (RDMA > Socket)
>
> In this project we ran 2× H100 nodes, each with 1 GPU, using `torchrun` with `--nnodes=2 --nproc_per_node=1`. NCCL initialized over Ethernet (`eth0`) because InfiniBand was not available in the Kubernetes pod network. NCCL debug logs (`NCCL_DEBUG=INFO`) confirmed the transport selection: `NCCL INFO NET/Socket : Using [0]eth0:10.0.44.87`.

---

**Q: A customer's multi-node training job hangs at startup with no error. How do you diagnose it?**

> The first thing I check is the NCCL initialization phase — this is where cross-node communication is established, and a hang here is almost always a network or rendezvous failure. Concrete steps:
> 1. Set `NCCL_DEBUG=INFO` and `NCCL_DEBUG_SUBSYS=INIT,NET` to expose what transport NCCL is trying to use
> 2. Check if the head node IP (`MASTER_ADDR`) is reachable from all worker nodes on `MASTER_PORT` (29500 by default) — a firewall or Security Group rule is the most common cause
> 3. Verify all ranks can resolve each other's IPs — DNS or `/etc/hosts` issues in Kubernetes pod networks surface here
> 4. Check if `dist.init_process_group()` was called before any CUDA operation — out-of-order initialization causes silent hangs
>
> In this project, SkyPilot provides `SKYPILOT_NODE_IPS` which makes the rendezvous straightforward. In a customer environment without an orchestrator, this step is often missing and must be scripted explicitly.

---

**Q: What is NCCL and why does it matter for GPU training at scale?**

> NCCL (NVIDIA Collective Communications Library) is NVIDIA's implementation of collective operations (all-reduce, broadcast, scatter, gather) optimized for GPU-to-GPU communication. It matters because:
> - It automatically selects the fastest available transport: NVLink (within a node) → InfiniBand RDMA → RoCE → TCP Socket — each an order of magnitude difference in bandwidth
> - All-reduce is the bottleneck in DDP: gradient synchronization happens every step, so communication latency directly caps throughput
> - At scale (hundreds of GPUs), naive all-reduce becomes O(n) — NCCL implements ring-allreduce and tree-reduce to keep it O(1) per node regardless of cluster size
>
> For a customer sizing H100 clusters, I'd ask: what's their inter-node network? If it's plain Ethernet without RoCE, they will be network-bound well before they're compute-bound. Nebius H100 SXM nodes on InfiniBand change this equation significantly.

---

### Kubernetes & Cloud Infrastructure

**Q: Explain the container image pull authentication problem you encountered and how you resolved it architecturally.**

> This is a common enterprise pattern problem. The symptom was `ErrImagePull: 403 Forbidden` on the Kubernetes nodes, despite local `docker push` succeeding.
>
> The root cause: **developer identity ≠ node identity**. When a developer runs `docker push`, they authenticate with their personal IAM credentials. Kubernetes nodes are separate compute principals — they authenticate independently to pull images. The developer's credentials don't transfer.
>
> There are three architectural solutions, from least to most production-ready:
> 1. **Public registry** — no auth needed, acceptable for open-source base images, unacceptable for proprietary model weights
> 2. **imagePullSecret** — a Kubernetes secret containing registry credentials, attached to the pod spec or default ServiceAccount. Limitations: IAM tokens are short-lived, secrets must be rotated, and orchestrators like SkyPilot may not expose a hook to inject them
> 3. **IAM Service Account on the node group** — the node's compute identity is granted `container-registry.images.pull` permission. No secrets to manage, no rotation, scales automatically. This is the production-grade approach and the one I would recommend to any Nebius customer deploying private training images.
>
> For this homework, public access was the pragmatic call. For a customer workload, I'd implement option 3 at cluster provisioning time.

---

**Q: A customer wants to deploy a multi-node GPU training pipeline on Nebius. What are the first five questions you ask in a discovery call?**

> 1. **What model and framework?** (PyTorch DDP, DeepSpeed, Megatron-LM, JAX?) — dictates parallelism strategy, NCCL requirements, and whether tensor/pipeline parallelism is needed alongside data parallelism
> 2. **What's the dataset size and where does it live?** — determines storage architecture: object storage with streaming vs. high-throughput NFS vs. local NVMe; this often dominates I/O-bound training
> 3. **What does their current MLOps stack look like?** — experiment tracking (MLflow, W&B), job scheduling (Slurm, Kubernetes, SkyPilot), artifact management — integration points that affect the solution design
> 4. **What is their target training time and cost envelope?** — determines GPU count, node count, whether spot/preemptible instances are viable, and whether the architecture needs checkpoint/resume logic
> 5. **Do they need inference after training, and at what scale?** — if yes, design the training pipeline to produce a deployment-ready artifact (quantized, compiled) and plan the transition to Nebius Endpoints, avoiding a second migration project

---

### SkyPilot & MLOps

**Q: What is SkyPilot and how does it fit into a customer's MLOps architecture?**

> SkyPilot is an open-source framework that abstracts multi-cloud and multi-cluster job orchestration. It sits between the developer's workstation and the underlying infrastructure (Kubernetes, VMs, cloud APIs) and handles:
> - Resource provisioning and selection (finds the cheapest available GPU that meets requirements)
> - Job lifecycle management (launch, monitor, log streaming, teardown)
> - Workdir sync (uploads local code to the API server, which distributes it to nodes)
>
> In a Nebius context, SkyPilot connects to mk8s clusters via kubeconfig and schedules pods using its own pod spec generation. This is powerful — a data scientist can submit a training job with a 50-line YAML without knowing Kubernetes — but the abstraction has limits. Operations that require direct pod spec control (custom tolerations, imagePullSecrets, node affinity) need to be handled at the infrastructure layer, not the SkyPilot layer.
>
> Architecturally, SkyPilot is best positioned as a **developer-facing job submission layer** sitting on top of a platform team's managed Kubernetes infrastructure. The platform team owns the cluster, IAM, and networking; the ML team owns the task YAML.

---

**Q: How would you design a reproducible distributed training setup for a customer that needs to onboard 10 ML engineers quickly?**

> The goal is to eliminate the 77% debug overhead we saw in this project. I'd design a three-layer platform:
>
> **Layer 1 — Infrastructure (owned by platform team):**
> Terraform or Nebius IaC to provision mk8s clusters with GPU node groups, IAM service accounts pre-attached, and private registry access granted. Engineers get a cluster that works on day one without touching auth.
>
> **Layer 2 — Job template library (owned by ML platform team):**
> A git repo of validated SkyPilot YAML templates for common patterns: single-node training, multi-node DDP, inference serving. Engineers clone and parameterize, they don't write from scratch. Context names, resource specs, and NCCL debug flags are pre-filled correctly.
>
> **Layer 3 — Developer workflow (owned by individual engineers):**
> A Makefile or CLI wrapper with targets: `make build`, `make push`, `make train`, `make logs`. Each target encodes the correct command with the correct flags (`--tail 0`, personal cluster name, registry URL format). Engineers run four commands, not forty.
>
> This architecture converts a ~120-minute first-run experience into a ~15-minute one for every subsequent engineer who onboards.

---

### Presales & Solution Design

**Q: How do you explain the value of Nebius to a CTO who is already using AWS or GCP for AI training?**

> I lead with three points:
>
> **1. Purpose-built for AI, not retrofitted.** AWS and GCP built general-purpose clouds and added GPU capabilities. Nebius was designed ground-up for AI/ML workloads — H100 SXM clusters with InfiniBand, NCCL-optimized networking, and integrated MLOps tooling (SkyPilot managed service, MLflow, container registry) are first-class citizens, not afterthoughts.
>
> **2. Cost structure.** Hyperscalers price GPU capacity as a premium on top of a general cloud platform. Nebius' pricing reflects a focused infrastructure model without the overhead of thousands of non-AI services. For a customer running continuous GPU training, this delta compounds fast.
>
> **3. No lock-in at the ML layer.** Nebius supports standard open-source tooling — SkyPilot, Kubernetes, PyTorch, HuggingFace. A customer's training code runs identically on Nebius and anywhere else. The value is in the infrastructure performance and price, not in proprietary APIs that create switching costs.
>
> Then I ask: "What does a single H100 node-hour cost you today, and what's your monthly GPU spend?" — that anchors the conversation in business terms.

---

**Q: A customer's PoC is failing. Their team is frustrated and considering going back to their existing provider. What do you do?**

> First, I separate signal from noise: is this a technical problem, a perception problem, or a scope mismatch?
>
> In this project, 77% of time was debugging — most of it caused by preventable configuration issues, not platform limitations. A frustrated customer in a PoC is often experiencing the same thing: they're measuring the onboarding friction, not the platform performance.
>
> My approach:
> 1. **Join a working session, don't just send docs.** Debug alongside their engineers. The fastest way to recover trust is to unblock them in real time.
> 2. **Isolate the platform from the configuration.** If their training job hangs, is it NCCL misconfiguration (their YAML) or a network issue (platform)? Reproduce the problem in a minimal environment to answer this definitively.
> 3. **Reframe the metrics.** If the training throughput number is what they care about, make sure we're measuring GPU utilization and samples/sec, not wall-clock time that includes misconfiguration overhead.
> 4. **Escalate early if it's a platform issue.** If I find a genuine platform limitation, I say so immediately and bring in Engineering. Hiding a real problem costs far more trust than surfacing it.
>
> The goal is not to win the PoC — it's to give the customer accurate information to make a good decision. That approach builds the long-term relationship, even if it means losing a short-term deal.

---

### Storytelling & Communication

**Q: How would you explain distributed GPU training to a Head of Finance who needs to approve a $500K cloud budget?**

> "Training a large AI model is like writing a very long book with a team. If one person writes it alone, it takes years. If you split the chapters across 100 writers working in parallel, it takes weeks — but they need to synchronize after every chapter so the story stays consistent.
>
> GPUs are the writers. The synchronization step is what NCCL handles over the network. The faster the network between GPUs, the less time is wasted waiting for synchronization, and the more efficiently your budget converts into trained model.
>
> The $500K you're approving funds the GPU hours to run that parallel process. The reason we're choosing Nebius over alternatives is that their H100 clusters have InfiniBand networking — the equivalent of giving your 100 writers a very fast intercom instead of email. The training job that takes 10 days on a slower network takes 6 days on Nebius. At $X per GPU-hour, that's a direct cost saving."

---

### Self-Assessment

**Q: What would you do differently if you ran this project again from scratch?**

> Three things:
>
> 1. **Provision infrastructure with IaC before writing a single line of training code.** All five naming and auth issues (Issues 1, 3, 4, 5, 7) were infrastructure problems that blocked the ML work entirely. A 30-minute IaC script at the start eliminates 65 minutes of debugging later.
>
> 2. **Set registry access policy at creation time, not as a fix.** The decision of public vs. private registry is an architectural decision with security implications. It should be made consciously upfront, not discovered under pressure when pods can't pull images.
>
> 3. **Validate each layer independently before stacking them.** Before running `sky launch`, verify: (a) `kubectl get nodes` works, (b) `docker pull <image>` from a node works, (c) `sky check kubernetes` returns the right context. Each layer validated in isolation means a failure in `sky launch` points to exactly one cause. Skipping this turns a 2-minute fix into a 30-minute debug session.
