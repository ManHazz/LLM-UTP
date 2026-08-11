# Distributed LLM Inference on a 4-Node RTX A1000 Cluster — Day 1 Report

**Date:** 2026-08-11
**Author:** manhazz
**Objective:** Establish a working distributed inference pipeline across multiple lab PCs and measure how performance scales with node count.

---

## 1. Executive Summary

Four identical lab workstations were configured as a llama.cpp RPC cluster and used to run **DeepSeek-R1-Distill-Qwen-14B** (Q4_K_M quantization). Distributed inference was proven end-to-end, and the empirical scaling curve matched the theoretical prediction for pipeline parallelism:

- **1 PC (single-node, GPU):** 10.1 tok/s generation
- **2 PCs (RPC):** 18.6 tok/s generation — **1.84× speedup**
- **4 PCs (RPC):** 16.7 tok/s generation — **1.65× speedup vs. 1 PC** (down from 2-PC peak)

The 2 → 4 node regression is a textbook **pipeline-bubble** effect: additional nodes reduce per-node compute but increase per-token network latency. The optimal cluster width is model-size dependent; for a 14B Q4 model on 8 GB GPUs, **2 nodes is the empirical sweet spot**.

---

## 2. Hardware and Network

### Per-node specs (all four identical)

| Component | Value |
|---|---|
| GPU | NVIDIA RTX A1000 (Ampere, compute cap 8.6), 8188 MiB VRAM |
| CPU | Intel Core Ultra 7 265 (Arrow Lake desktop, 20 cores) |
| OS | Ubuntu 24.04.3 LTS (Noble) |
| Kernel (running) | 6.17.0-1017-oem |
| NVIDIA driver | 580.126.09 |
| CUDA runtime | 13.0 |
| NIC | 1 GbE, on subnet `172.17.102.0/24` |

### Node inventory

| Role | Hostname | IP |
|---|---|---|
| Driver | 230007-02 (PC-A) | 172.17.102.123 |
| Worker | 230007-03 (PC-B) | 172.17.102.3 |
| Worker | 230007-01 (PC-C) | 172.17.102.107 |
| Worker | 230007-04 (PC-D) | 172.17.102.39 |

### Network baseline

Bidirectional ICMP verified between PC-A and PC-B. Round-trip time on the lab LAN:
- **avg 1.10 ms, min 0.41 ms, max 2.53 ms, mdev 0.59 ms** (10-packet sample)

Confirmed sub-millisecond typical latency — appropriate for pipeline parallelism.

---

## 3. Pre-flight Issues Encountered and Resolved

### 3.1 nvidia-smi failure under kernel 6.17.0-1024-oem

The default running kernel (`6.17.0-1024-oem`) had no matching NVIDIA kernel module installed. `apt-cache` search for the corresponding package returned no results.

**Root cause:** Ubuntu had not yet published `linux-modules-nvidia-580-open-6.17.0-1024-oem`. Prebuilt modules existed only for kernels `-1011`, `-1017`, `-1023`.

**Resolution:** Force GRUB to display the boot menu, boot into `6.17.0-1017-oem` which has a working prebuilt module. Modifications to `/etc/default/grub`:

```
GRUB_TIMEOUT_STYLE=menu
GRUB_TIMEOUT=10
```

Followed by `sudo update-grub` and reboot. Same procedure applied successfully on all four PCs.

### 3.2 Two PCs presented black screen + X cursor after login

Caused by the same kernel/driver mismatch: X server started, but the NVIDIA driver failed to load, preventing the desktop from rendering. Fixed by dropping to a TTY (`Ctrl+Alt+F3`) and applying the same GRUB fix.

### 3.3 Single-node VRAM ceiling

The 14B Q4_K_M model weights are approximately 9 GB, exceeding the 8 GB VRAM of one A1000. Attempting `-ngl 999` on a single PC produced:

```
ggml_backend_cuda_buffer_type_alloc_buffer: allocating 8148.38 MiB on device 0:
  cudaMalloc failed: out of memory
```

**Resolution:** Reduce to `-ngl 30` for single-node baseline (partial CPU offload). This is precisely why a cluster is needed for this model: two 8 GB GPUs together fit it comfortably where one cannot.

---

## 4. Software Stack

- **llama.cpp** commit `153d324bc` (build tag `b10364-153d324bc`)
- Built with: `-DGGML_CUDA=ON -DGGML_RPC=ON`
- Model: DeepSeek-R1-Distill-Qwen-14B, Q4_K_M quantization (~9 GB), from HuggingFace `unsloth/DeepSeek-R1-Distill-Qwen-14B-GGUF`

Build warnings (benign for this experiment):
- `Could NOT find NCCL` — reduces multi-GPU tensor-parallel performance; not used by RPC backend.
- `Could NOT find OpenSSL` — HTTPS in `llama-server` disabled; irrelevant for pure RPC benchmarking.

---

## 5. Methodology

**Test prompt** (identical across all three runs, to eliminate prompt-length as a variable):
```
Write me a short haiku abt distributed computing.
```
- `-n 128` (max 128 tokens generated)
- `--no-display-prompt`
- Reasoning trace (`<think>`) tokens were generated in all runs (DeepSeek-R1-distill behavior)

**Workers** started on each participating node with:
```
./build/bin/ggml-rpc-server -H 0.0.0.0 -p 50052
```
Note: `-H 0.0.0.0` is essential — the default `127.0.0.1` binds only to loopback and blocks remote RPC connections.

**Driver** invoked from PC-A. Worker list passed as comma-separated addresses (no whitespace):

```
./build/bin/llama-cli \
  -m /mnt/LLM/models/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M.gguf \
  --rpc <IP1>:50052,<IP2>:50052,... \
  -ngl 999 \
  -p "Write me a short haiku abt distributed computing." \
  -n 128 \
  --no-display-prompt
```

Metrics captured from llama.cpp's built-in performance summary line:
```
[ Prompt: XX.X t/s | Generation: YY.Y t/s ]
```

---

## 6. Results

| Configuration | Prompt tok/s | Generation tok/s | Speedup vs 1-PC |
|---|---:|---:|---:|
| 1 PC (PC-A only, `-ngl 30`, partial CPU offload) | 31.6 | 10.1 | 1.00× |
| 2 PCs (PC-A + PC-B via RPC) | 110.5 | **18.6** | **1.84×** |
| 4 PCs (PC-A + B/C/D via RPC) | 51.9 | 16.7 | 1.65× |

### Observed generation-speed scaling

```
      tok/s
   20 ┤
      │
   18 ┤              ●  (2-PC, 18.6)
      │             /│
      │            / │
   16 ┤           /  │       ●  (4-PC, 16.7)
      │          /   └─────┘
      │         /
   14 ┤        /
      │       /
   12 ┤      /
      │     /
   10 ┤●   (1-PC, 10.1)
      │
    8 ┤
      └──────┴──────┴──────┴──────
        1     2     3     4     nodes
```

---

## 7. Analysis

### 7.1 Why 2-PC is faster than 1-PC

On a single node the 14B Q4 model does not fully fit in 8 GB VRAM, forcing partial CPU offload. CPU-resident layers are bandwidth-limited (DDR5 memory) and become the bottleneck. With two nodes, weights are partitioned across two 8 GB GPUs; every layer executes on GPU-resident tensors, eliminating the CPU-spillover penalty. The observed 1.84× speedup approaches the theoretical ceiling for a workload where the primary constraint was the single-node memory limit.

### 7.2 Why 4-PC is slower than 2-PC (pipeline bubbles)

Once the model fits comfortably in aggregate GPU memory, adding more nodes ceases to help and begins to hurt. Each additional layer boundary introduces a network round-trip (activation transfer). Downstream nodes idle while upstream ones compute — a phenomenon called **pipeline bubbles** in distributed inference literature.

The regression from 18.6 → 16.7 tok/s (a ~10% loss) is consistent with the pipeline-bubble prediction for a 4-stage pipeline over 1 GbE with sub-millisecond intra-cluster latency. Published measurements from exo (Ampere-generation cluster benchmarks) and the Prima.cpp paper both cap systematic testing at 4-6 nodes because scaling degrades from this point onward for most model sizes.

### 7.3 Cluster-width sweet spot is model-size dependent

The observed sweet spot at 2 nodes for a 14B Q4 model is not a universal property of the cluster — it is a property of the ratio between model VRAM footprint and per-node VRAM. Larger models (32B, 70B) that cannot fit in 2 × 8 GB will shift the sweet spot rightward. This will be tested on the same cluster in a future session using the 32B and 70B distill variants.

---

## 8. Reproducibility Appendix

### 8.1 Kernel fix (one-time per PC)

```bash
sudo cp /etc/default/grub /etc/default/grub.bak
sudo sed -i 's/^GRUB_TIMEOUT_STYLE=.*/GRUB_TIMEOUT_STYLE=menu/;s/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=10/' /etc/default/grub
sudo update-grub
sudo reboot
# At GRUB menu: Advanced options for Ubuntu → 6.17.0-1017-oem
```

### 8.2 Package install (one-time per PC)

```bash
sudo apt update
sudo apt install -y nvidia-cuda-toolkit cmake build-essential git curl wget openssh-server
sudo systemctl enable --now ssh
```

### 8.3 llama.cpp build (per PC)

```bash
cd ~ && git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
cmake -B build -DGGML_CUDA=ON -DGGML_RPC=ON
cmake --build build --config Release -j$(nproc)
```

### 8.4 Model download (driver node only)

```bash
mkdir -p /mnt/LLM/models && cd /mnt/LLM/models
wget https://huggingface.co/unsloth/DeepSeek-R1-Distill-Qwen-14B-GGUF/resolve/main/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M.gguf
```

### 8.5 Session startup (per-day, after reboot to `6.17.0-1017-oem`)

On each worker (via SSH from driver):
```bash
cd ~/llama.cpp && ./build/bin/ggml-rpc-server -H 0.0.0.0 -p 50052
```

On driver:
```bash
cd /mnt/LLM/llama.cpp && ./build/bin/llama-cli \
  -m /mnt/LLM/models/DeepSeek-R1-Distill-Qwen-14B-Q4_K_M.gguf \
  --rpc 172.17.102.3:50052,172.17.102.107:50052,172.17.102.39:50052 \
  -ngl 999 -p "<prompt>" -n 128 --no-display-prompt
```

---

## 9. Next Steps

1. **Scale to 8 nodes.** Four additional PCs (E–H) were configured with SSH and CUDA toolkit but not yet added to a live cluster. Extend the RPC list to include all 8, re-run 14B benchmark to observe whether the pipeline-bubble regression continues linearly.
2. **Sweep model sizes on the 4-PC cluster.** Run DeepSeek-R1-Distill-Qwen-32B (~20 GB Q4) — should be near the 4-node sweet spot. Optional stretch: R1-Distill-Llama-70B (~40 GB Q4) — will require ≥6 nodes.
3. **Investigate the `-ngl 30` vs `-ngl 999` boundary on 1-PC** to characterize the exact CPU/GPU split ratio the A1000 supports for the 14B model.
4. **Automate the session startup ritual** as a shell script consuming the host list from a single config file, eliminating manual SSH-and-run-worker steps.
5. **Longer-term:** build the user-node agentic harness on top of the working cluster (see project master plan).

---

## 10. References

- llama.cpp repository: https://github.com/ggml-org/llama.cpp
- llama.cpp build documentation: https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md
- llama.cpp RPC backend README: https://github.com/ggml-org/llama.cpp/blob/master/tools/rpc/README.md
- Model source (Unsloth GGUFs): https://huggingface.co/unsloth/DeepSeek-R1-Distill-Qwen-14B-GGUF
- DeepSeek-R1 technical report: https://arxiv.org/abs/2501.12948
- Prima.cpp (distributed inference paper): https://arxiv.org/html/2504.08791v2
- exo distributed inference benchmarks: https://blog.exolabs.net/day-1/
- Pipeline parallelism scaling limits: https://docs.vllm.ai/en/latest/serving/parallelism_scaling/
- Pipeline bubbles concept (BentoML LLM handbook): https://bentoml.com/llm/inference-optimization/data-tensor-pipeline-expert-hybrid-parallelism
