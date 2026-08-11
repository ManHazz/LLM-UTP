# Session 2 Plan — 2D Sweep: Cluster Size × Model Size

**Date planned:** 2026-08-11 (executed 2026-08-12)
**Objective:** Empirically map the sweet-spot curve — for every N-node cluster size, which model size runs fastest? Deliverable is a 2D table + a chosen "optimal (N, model)" allocation for the lab.
**Stretch objective:** If time permits, prototype orchestrated sub-clustering (two independent 4-node clusters behind a router).

---

## 0. Success criteria — what "done" looks like

- [ ] All 8 PCs reachable via SSH from PC-A, all with llama.cpp CUDA+RPC build
- [ ] At least 4 model sizes benchmarked across 1, 2, 4, 6, 8 node counts (minimum viable sweep)
- [ ] Results table populated in `report.md`
- [ ] Sweet-spot ridge identified for each model
- [ ] Executive summary written

Stretch:
- [ ] Two 4-node sub-clusters running simultaneously with different requests
- [ ] Simple round-robin router forwarding to either cluster

---

## 1. Pre-session — do at home tonight (~15 min)

- [ ] List all model files needed and check disk space math:
  - Qwen2.5-1.5B Q4 ≈ 0.9 GB
  - Qwen2.5-3B Q4 ≈ 1.8 GB
  - Qwen2.5-Coder-7B Q4 ≈ 4.4 GB
  - **R1-Distill-Qwen-14B Q4** ≈ 8.6 GB (already have)
  - **R1-Distill-Qwen-32B Q4** ≈ 19.9 GB (download in progress or ready)
  - **R1-Distill-Llama-70B Q4** ≈ 42.5 GB
  - **Total ≈ 78 GB.** Verify `/mnt/LLM/models/` has ~80 GB free tomorrow morning.
- [ ] Confirm the plan with yourself: does the sweep grid below match what you actually want to measure?
- [ ] Print or bookmark this file so you don't context-switch to laptop mid-experiment.

---

## 2. Session startup ritual (~30 min at the lab)

### 2.1 Boot every PC to the working kernel

For each of PC-A through PC-H, at the console:
- Power on → GRUB menu appears (thanks to the sed fix from session 1) → **Advanced options** → **6.17.0-1017-oem** → Enter.

If any PC boots straight to `-1024-oem` (default reverted or GRUB fix lost): repeat the sed fix from `11-8/hardware_notes.txt` section 3.1.

### 2.2 Verify each is alive

From PC-A:
```bash
for h in pcb pcc pcd; do
  ssh $h "uname -r; nvidia-smi -L"
done
```

Every one should print `6.17.0-1017-oem` and `GPU 0: NVIDIA RTX A1000 ...`.

### 2.3 Bring PCs E-H online (SSH keys + build)

If PCs E-H were prepped in session 1 (CUDA installed, openssh-server installed):

```bash
# Push SSH keys:
ssh-copy-id user@172.17.102.99    # PC-E
ssh-copy-id user@172.17.102.1     # PC-F  (yes really .1)
ssh-copy-id user@172.17.102.100   # PC-G
ssh-copy-id user@172.17.102.131   # PC-H

# Add to ~/.ssh/config on PC-A:
cat >> ~/.ssh/config <<'EOF'

Host pce
    HostName 172.17.102.99
    User user
Host pcf
    HostName 172.17.102.1
    User user
Host pcg
    HostName 172.17.102.100
    User user
Host pch
    HostName 172.17.102.131
    User user
EOF

# Verify each:
for h in pce pcf pcg pch; do
  ssh $h "hostname; nvidia-smi -L; uname -r"
done
```

Then build llama.cpp on all four in parallel:
```bash
for h in pce pcf pcg pch; do
  ssh $h 'cd ~ && git clone https://github.com/ggml-org/llama.cpp 2>/dev/null; \
    cd ~/llama.cpp && \
    cmake -B build -DGGML_CUDA=ON -DGGML_RPC=ON 2>&1 | tail -3 && \
    cmake --build build --config Release -j$(nproc) 2>&1 | tail -3' > ~/build-log-$h.txt 2>&1 &
done
wait
echo "==== all E-H builds finished ===="
```

Confirm:
```bash
for h in pce pcf pcg pch; do
  echo "=== $h ==="
  ssh $h "ls ~/llama.cpp/build/bin/ggml-rpc-server ~/llama.cpp/build/bin/llama-cli 2>&1"
done
```

### 2.4 Download missing model files (concurrent with 2.3 above)

On PC-A, kick off in a separate terminal:
```bash
cd /mnt/LLM/models

# Small models (fast):
wget -c https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1.5b-instruct-q4_k_m.gguf
wget -c https://huggingface.co/Qwen/Qwen2.5-3B-Instruct-GGUF/resolve/main/qwen2.5-3b-instruct-q4_k_m.gguf
wget -c https://huggingface.co/Qwen/Qwen2.5-7B-Instruct-GGUF/resolve/main/qwen2.5-7b-instruct-q4_k_m.gguf

# Bigger:
wget -c https://huggingface.co/unsloth/DeepSeek-R1-Distill-Qwen-32B-GGUF/resolve/main/DeepSeek-R1-Distill-Qwen-32B-Q4_K_M.gguf
wget -c https://huggingface.co/unsloth/DeepSeek-R1-Distill-Llama-70B-GGUF/resolve/main/DeepSeek-R1-Distill-Llama-70B-Q4_K_M.gguf
```

`-c` = continue if partially downloaded. Safe to Ctrl-C and rerun.

---

## 3. The 2D sweep — main event

### 3.1 Sweep grid

| Model | 1 node | 2 nodes | 4 nodes | 6 nodes | 8 nodes |
|---|---|---|---|---|---|
| Qwen-1.5B | | | | | |
| Qwen-3B | | | | | |
| Qwen-7B | | | | | |
| R1-Distill-14B | 10.1* | 18.6* | 16.7* | | |
| R1-Distill-32B | | | | | |
| R1-Distill-70B | | | | | |

*Values from session 1 — do NOT rerun unless you have time to spare.*

Grid dimensions: 6 models × 5 node counts = 30 cells. Session 1 already covers 3 cells. **27 cells to run.**

**Time budget per cell:**
- Model load: 15-45 sec (depends on size)
- Warmup + measurement: 30-60 sec
- Teardown between runs: 5 sec
- Total per cell: **~90 sec**

27 × 90 sec = **~40 min pure benchmark time.** Add worker restart overhead between node-count changes = ~1.5 hours realistic.

### 3.2 Command template

**Workers** (start on each participating node BEFORE running the driver):

In each worker pane (SSH'd to the target PC):
```bash
cd ~/llama.cpp
./build/bin/ggml-rpc-server -H 0.0.0.0 -p 50052
# Leave running. Ctrl-C when done with this node-count level.
```

**Driver — use `llama-bench` for standardized output** (cleaner than parsing llama-cli):

```bash
cd /mnt/LLM/llama.cpp

# Example: 4-PC run on the 14B model
./build/bin/llama-bench \
  -m /mnt/LLM/models/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M.gguf \
  --rpc 172.17.102.3:50052,172.17.102.107:50052,172.17.102.39:50052 \
  -ngl 999 \
  -p 128 -n 128 \
  -r 3
```

Key flags:
- `-p 128 -n 128` — prompt-eval length + generation length. Bench does both.
- `-r 3` — repeat 3 times, report average and stddev.
- Output is a nice pipe-delimited table.

**Test with two prompt/gen lengths per cell** to capture the short-vs-long behavior:

```bash
# Short-answer:
-p 64 -n 128

# Long-answer:
-p 512 -n 1024
```

That doubles cell count (60 cells) but the sweep becomes ~3 hours and produces a genuinely defensible dataset.

### 3.3 IP list quick reference

Copy-paste-ready RPC target strings:

- **2 PC (A+B):** `172.17.102.3:50052`
- **3 PC:** `172.17.102.3:50052,172.17.102.107:50052`
- **4 PC:** `172.17.102.3:50052,172.17.102.107:50052,172.17.102.39:50052`
- **6 PC:** `172.17.102.3:50052,172.17.102.107:50052,172.17.102.39:50052,172.17.102.99:50052,172.17.102.1:50052`
- **8 PC:** `172.17.102.3:50052,172.17.102.107:50052,172.17.102.39:50052,172.17.102.99:50052,172.17.102.1:50052,172.17.102.100:50052,172.17.102.131:50052`

(Note: PC-A is always the driver — its own GPU contributes local layers, so the `--rpc` list has N-1 entries for an N-node cluster.)

### 3.4 Data capture

**Suggested:** pipe every `llama-bench` invocation to a per-cell CSV-friendly log:

```bash
./build/bin/llama-bench ... -o csv >> ~/sweep-results.csv
```

`-o csv` outputs one line per test, comma-separated. Grows as you go. At end of session, one file has everything.

Alternative low-tech: keep a spreadsheet open on your phone, fill each cell manually. Slower but you notice anomalies in real time.

### 3.5 Known failure modes to log, not chase

- **OOM at high `-ngl` on small clusters + big models** — expected. Note the (N, model) cell as "OOM at -ngl 999" and move on. Don't try to optimize partial-offload today.
- **Model won't load at all on 1-node for models > 8 GB** — expected. Cell = "N/A, insufficient VRAM."
- **Worker crashes mid-run** — restart it, retry the cell. If it crashes twice, log and skip.

---

## 4. Stretch goal — orchestrated sub-clustering

**Only start if step 3 is done AND you have ≥45 min left in the lab.**

### 4.1 Setup

Split the 8 nodes into two independent 4-node clusters:
- **Sub-cluster A:** PC-A (driver) + PC-B + PC-C + PC-D as workers
- **Sub-cluster B:** PC-E (driver) + PC-F + PC-G + PC-H as workers

Each runs `llama-server` (not `llama-cli`) so they present an HTTP API:

**On PC-A (sub-cluster A driver):**
```bash
cd /mnt/LLM/llama.cpp
./build/bin/llama-server \
  -m /mnt/LLM/models/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M.gguf \
  --rpc 172.17.102.3:50052,172.17.102.107:50052,172.17.102.39:50052 \
  -ngl 999 \
  --host 0.0.0.0 --port 8080
```

**On PC-E (sub-cluster B driver):**
```bash
cd ~/llama.cpp
./build/bin/llama-server \
  -m ~/models/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M.gguf \
  --rpc 172.17.102.1:50052,172.17.102.100:50052,172.17.102.131:50052 \
  -ngl 999 \
  --host 0.0.0.0 --port 8080
```

(You'll need to copy the model to PC-E first — `scp /mnt/LLM/models/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M.gguf pce:~/models/`)

### 4.2 Minimal orchestrator (bash, ~10 lines)

On a spare terminal on PC-A (or any node), rotate requests:

```bash
#!/bin/bash
# rotate-router.sh — trivial round-robin between two sub-clusters
CLUSTERS=("172.17.102.123:8080" "172.17.102.99:8080")
i=0
prompt="Write a short poem about $1"

while true; do
  target="${CLUSTERS[$((i % 2))]}"
  echo "→ dispatching to $target"
  curl -s "http://$target/completion" \
    -H "Content-Type: application/json" \
    -d "{\"prompt\":\"$prompt\", \"n_predict\":128}" | jq -r '.content'
  ((i++))
  read -p "next? " -r
done
```

### 4.3 Measurement — the point of the exercise

Run 4 requests simultaneously via the router. Compare:
- Total wall time for 4 requests through the router (should be ~2× single-cluster response time, because 2 clusters serve 2 at a time in parallel)
- Total wall time for 4 requests through a single 8-node cluster (should be ~4× single-cluster response time, because pipeline serves them serially)

**Expected result:** the sub-cluster architecture is ~2× faster under concurrent load, even though each individual request is *no faster* than in a single 4-node cluster.

This is the empirical demonstration of the framing correction from the plan discussion: **sub-clustering wins on throughput/concurrency, not per-request latency.**

### 4.4 Data to capture

- Time for 4 sequential single-request queries against 1 sub-cluster
- Time for 4 simultaneous queries against the router (2 sub-clusters)
- Time for 4 simultaneous queries against a single 8-node cluster

Three numbers. Report goes in the day-2 report.md.

---

## 5. End of session — before leaving

- [ ] Save `sweep-results.csv` to `/mnt/LLM/12-8/` (or wherever, but path known)
- [ ] Ctrl-C all `ggml-rpc-server` on workers
- [ ] Copy of results file to laptop (paste service or USB, per session 1 procedure)
- [ ] Note any anomalies or things to investigate in a `blockers.txt` file

---

## 6. Post-session — write up

Use `/mnt/projects/LLM/report-template.md` as the skeleton. Fill in:
- Section 5 (Results): the populated 2D table + optional CSV export
- Section 6 (Analysis): identify sweet-spot ridge, compare to session-1 result at the (14B, 2-node) cell — should match ~18.6 t/s
- Section 8 (Next Steps): whatever the sweep reveals — e.g. "70B is the biggest model that fits 8 nodes with margin, so focus the user-node harness plan on serving 70B"

Save the finished report to `/mnt/projects/LLM/12-8/report.md`.

---

## 7. If everything goes sideways

Priority ordering for "cut scope":
1. Drop the two-prompt-length variant (`-p 512 -n 1024`) — halves runtime.
2. Drop the smallest models (1.5B, 3B) — they aren't the interesting data.
3. Drop the 6-node cluster row — 4 and 8 bracket it.
4. Drop the 70B row if 32B is where scaling clearly saturates.
5. Skip the sub-cluster stretch entirely.

Minimum viable session: 4 models × 3 node counts (1, 4, 8) = 12 cells = ~20 min bench time. Still gives you a meaningful curve.
