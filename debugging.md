# Debugging Report — Nebius DDP Training Homework

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

## Detailed Issue Descriptions

### Issue 1 — Wrong flag for node-group creation
**Command:** `nebius mk8s node-group create --cluster-id ...`
**Error:** `unknown flag: --cluster-id`
**Cause:** Incorrect flag name — the Nebius CLI uses `--parent-id` to reference the parent cluster.
**Fix:** Use `--parent-id` instead of `--cluster-id`.

---

### Issue 2 — Shared SkyPilot cluster contention
**Error:** `Cluster 'ddp-run' was created by another user. Blocked by other requests.`
**Cause:** All students shared the same `ddp-run` cluster name on the shared SkyPilot API server, causing a queue.
**Fix:** Used a personal cluster name `ariel-ddp` with `sky launch -c ariel-ddp`.

---

### Issue 3 — Wrong Kubernetes context name in train_job.yaml
**Error:** SkyPilot couldn't resolve the cluster context `k8s/ariel-mitiushkin-mk8s`.
**Cause:** SkyPilot registers the cluster under its full internal name, not the display name.
**Fix:** Updated `infra` to the full context name:
```
k8s/nebius-mk8s-ariel-mitiushkin-mk8s-e00qn3h3n6rqax9gr2
```

---

### Issue 4 — ErrImagePull 403 Forbidden (shared registry)
**Error:** `403 Forbidden` pulling `cr.eu-north1.nebius.cloud/e00g98qe9694ryna8e/nebius-trainer:v3`
**Cause:** The initial `train_job.yaml` pointed at a shared registry the student's cluster had no IAM access to.
**Fix:** Created a personal registry and built/pushed a personal image.

---

### Issue 5 — Wrong registry ID format in URL
**Error:** `Entity Registry not found by id registry-e00xbrt95y1x46pwr3`
**Cause:** The Nebius registry URL uses only the bare numeric ID — the `registry-` prefix is a resource name, not part of the URL path.
**Fix:** Changed URL from:
```
cr.eu-north1.nebius.cloud/registry-e00xbrt95y1x46pwr3/nebius-trainer:v1
```
to:
```
cr.eu-north1.nebius.cloud/e00xbrt95y1x46pwr3/nebius-trainer:v1
```

---

### Issue 6 — Image not found (not yet pushed)
**Error:** `rpc error: code = NotFound — nebius-trainer:v1: not found`
**Cause:** The Docker image had not yet been built and pushed to the registry.
**Fix:** Ran from WSL project directory:
```bash
nebius registry configure-helper
docker build -t cr.eu-north1.nebius.cloud/e00xbrt95y1x46pwr3/nebius-trainer:v1 .
docker push cr.eu-north1.nebius.cloud/e00xbrt95y1x46pwr3/nebius-trainer:v1
```

---

### Issue 7 — 403 Forbidden on personal registry (Kubernetes pull auth)
**Error:** `403 Forbidden` pulling from personal registry `e00xbrt95y1x46pwr3`
**Cause:** The mk8s node group had no credentials to pull from a private Nebius Container Registry. An `imagePullSecret` approach was attempted but SkyPilot pods did not pick it up automatically.
**Fix:** Set the registry to **public access** in the Nebius console:
> Container Registry → registry → Settings → Public access → Enable

---

### Issue 8 — IAM token command not found
**Command:** `nebius iam get-token`
**Error:** `unknown command "get-token" for "nebius iam"`
**Cause:** Incorrect subcommand name; varies by CLI version.
**Fix:** Bypassed entirely by making the registry public (see Issue 7).

---

### Issue 9 — Incomplete training log (default 1000-line tail)
**Command:** `sky logs ariel-ddp > training_log.txt`
**Problem:** Only the last 1000 lines were written — the NCCL init section at the start of the log was missing.
**Fix:**
```bash
sky logs ariel-ddp --tail 0 > training_log.txt
```

---

### Issue 10 — Docker build location confusion (WSL vs Windows)
**Cause:** Unclear whether to run `docker build` in Windows PowerShell or the WSL terminal. The Nebius CLI and `sky` are installed only in WSL.
**Fix:** All commands (nebius, sky, docker) must run in the WSL terminal. Docker Desktop on Windows must have WSL2 integration enabled for the correct distro.

---

## Key Takeaways

- **~77% of total time was spent debugging** (~92 min) vs ~23% on actual execution (~27 min).
- The biggest time sink was **private registry authentication** (Issues 4, 7 — combined ~45 min). For homework environments, setting the registry to public access from the start saves significant time.
- **SkyPilot context names** are not the same as cluster display names — always verify with `sky check kubernetes`.
- **All tooling (nebius CLI, sky, docker) must run inside WSL** when working on Windows.
