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
