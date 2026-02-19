# Minimal Hardware Requirements Methodology (Repo-Agnostic)

Goal: produce a defensible “minimum hardware” spec for a repository under a **defined workload**.

---

## 1) Define the Target Workload (MVW)
Write down a **Minimum Verifiable Workload (MVW)** so “minimum” is meaningful.

**MVW checklist**
- One command (or script) that exercises the core path (e.g., run inference once, start service + 1 request, run a tiny training step, run smoke tests).
- Fixed input scale: batch size, concurrency, dataset subset size, sequence length, image resolution, etc.
- Clear success criteria: exit code 0, specific log line, artifact produced, endpoint returns 200.

**Output**
- `mvw_command: ...`
- `input_scale: ...`
- `success_criteria: ...`

---

## 2) Separate Phases: Build vs Run
Minimum resources can differ a lot.

- **Build/Install phase**: compiling deps, pulling images, building extensions
- **Run phase**: the actual workload

**Output**
- `build_min_spec` (optional)
- `run_min_spec` (primary)

---

## 3) Static Triage (Hard Requirements)
Before measuring, scan for “must-haves”.

Look for:
- GPU dependency: CUDA/ROCm mentions, `--gpus`, `nvidia/cuda` images, `torch+cu*`, `tensorflow-gpu`, JAX CUDA, etc.
- Large fixed assets: model weights, datasets, caches.
- Multiprocessing defaults: workers/threads that inflate RAM.
- OS/ISA constraints: AVX2/AVX512, specific kernel/driver requirements.

**Output**
- `requires_gpu: yes/no`
- `likely_bottlenecks: RAM/VRAM/Disk/CPU/Network`

---

## 4) One-Pass Profiling (No Search / No Binary Search)
Run the MVW once (or 3 times) on a “safe” machine and record **peak usage**.

Record:
- **Peak RAM (RSS)** during run
- **Peak VRAM** (if GPU)
- **Peak disk usage** (deps + data + caches + temp)
- **CPU saturation** (avg %, max threads), and runtime/latency

**Peak RAM: measure the whole process tree, not just the main process.**  
Tools like `/usr/bin/time -v` report only the **main process** RSS. If the MVW spawns many children (e.g. `go build`/`go test`, `make -j`, Maven, parallel test runners), total system RAM can be much higher. Prefer one of:
- **Process-tree RSS**: while the MVW runs, periodically sum RSS of the root process and all descendants (e.g. via `ps -eo pid,ppid,rss` and tree traversal), then take the maximum over the run.
- **Cgroup memory peak**: run the MVW in a cgroup and read `memory.max_usage_in_bytes` (cgroup v1) or the equivalent peak-usage stat after the run.

Use the **measured** peak (not the single-process number) as `peak_ram_gb`.

**Output**
- `peak_ram_gb`
- `peak_vram_gb` (optional)
- `peak_disk_gb`
- `cpu_utilization_summary`
- `runtime_or_latency`

---

## 5) Report Raw Peaks as Minimum Spec (No Margins)
Publish **raw measured peaks** only. Optionally round up to common SKUs for readability (e.g. 7.2 GB → 8 GB, 0.5 GB disk → 1 GB).

- **min_ram_gb**: smallest common SKU ≥ max(build_peak_ram, run_peak_ram) — e.g. SKUs 2, 4, 8, 16, 32 GB
- **min_disk_gb**: smallest common SKU ≥ peak_disk — e.g. 1, 2, 4, 8 GB
- **min_vcpu**: 1–2 vCPU for “can run”; or the smallest count that meets your time/latency target

No safety multipliers (no ×1.3, +1 GB, etc.); the reported minimum is the observed peak (rounded up to SKU if desired).

**Output**
- `min_ram_gb`
- `min_vram_gb` (optional)
- `min_disk_gb`
- `min_vcpu`

---

## 6) Scaling Rules (Optional, Still No Search)
If you need “minimum” at different input scales, model resources as:

- `RAM ≈ A_fixed + B * batch + C * concurrency + D * input_length`
- `VRAM ≈ W_weights + K * batch * input_length` (common for DL inference)

Measure:
1) **Idle after load** (fixed cost)
2) **One MVW step** (fixed + variable)

Then estimate for new scales without re-running everything.

---

## 7) Report Template (What You Publish)
Include:
- MVW definition (command, input scale, success criteria)
- Environment (OS, key dependency versions, CUDA/driver if relevant)
- **Peaks observed** (RAM/VRAM/Disk/CPU + runtime) — raw data, no margins
- Minimum spec = peaks rounded up to common SKUs (optional)
- Any knobs used to minimize resource usage (batch=1, workers=0/1, fp16, etc.)

Example summary:
- **Run-min**: `2 vCPU / 8GB RAM / 15GB disk / (optional) 1 GPU >= 12GB VRAM`
- MVW: `...`
- Peaks (raw): `RAM 5.2GB, Disk 9GB, VRAM 7.6GB`

---

## Common Pitfalls
- Default configs aren’t minimal (workers, concurrency, cache, batch).
- **Single-process RSS undercounts RAM**: for multi-process MVWs (build systems, test runners), `time -v` or similar gives only the parent's RSS; measure process-tree or cgroup peak to get a defensible minimum.
- `/dev/shm` too small can look like random hangs.
- Build-time requirements can exceed run-time requirements.
- Network/model downloads can dominate first run; separate “cold start” vs “steady state”.