# Lab Day Checklist — 40-PC LLM Project

**Date:** ____________________
**PCs tested:** PC-A ______  PC-B ______  PC-C ______

**Goal of the day:** replace every "worst-case assumption" in the architecture plan with a real measured number. By end of day you should be able to answer: *is this thing actually buildable on these PCs?*

Assumes Linux with sudo. Add 30-50% time if Windows / no admin.

---

## Track A — Hardware census (60-90 min)

Pick 3 PCs from *different tables* (not adjacent — you want a representative sample). For each PC, capture the following into a text file (e.g. `pc-A.txt`).

- [ ] **PC-A** — GPU: `nvidia-smi` (or `lspci | grep -i vga` if no NVIDIA)
- [ ] **PC-A** — RAM: `free -h`
- [ ] **PC-A** — CPU: `lscpu`  (or `wmic cpu get name` on Windows)
- [ ] **PC-A** — Disk: `df -h /`
- [ ] **PC-A** — OS: `cat /etc/os-release`  (or `winver`)
- [ ] **PC-A** — Admin/sudo: try `sudo -v`. Note exact restriction if denied.
- [ ] **PC-A** — Network: `ip a`  (or `ipconfig`) — record IP + subnet
- [ ] **Repeat all above for PC-B**
- [ ] **Repeat all above for PC-C**

### Critical fields — fill this table:

| Field | PC-A | PC-B | PC-C |
|---|---|---|---|
| GPU model | | | |
| VRAM | | | |
| CUDA compute cap | | | |
| Driver version | | | |
| Total RAM | | | |
| CPU model | | | |
| CPU cores/threads | | | |
| OS + version | | | |
| Sudo/admin? | | | |
| IP address | | | |

### Red flags to watch for
- [ ] **CUDA compute capability < 6.0** (pre-Pascal) — rules out most modern CUDA stacks. Plan pivots to CPU-primary everywhere.
- [ ] **No sudo / locked-down Windows** — every install goes through Docker Desktop / WSL2 (add friction).
- [ ] **RAM < 8 GB** — even the small user-node model will struggle; may need to drop from Qwen 3B to Qwen 1.5B.
- [ ] **PCs on different subnets** — big-guy cluster communication will be routed, adding latency.

---

## Track B — Network reality check (30-45 min)

Install `iperf3` on all 3 test PCs:
- [ ] `sudo apt install iperf3` (or download binary for Windows)

### Point-to-point throughput
- [ ] On PC-A: `iperf3 -s`
- [ ] On PC-B: `iperf3 -c <PC-A-IP> -t 30` → record: **______ Mbit/s**
- [ ] On PC-C: `iperf3 -c <PC-A-IP> -t 30` → record: **______ Mbit/s**

### Contention test
- [ ] Run **both** PC-B and PC-C as clients against PC-A simultaneously.
  - Aggregate throughput: **______ Mbit/s**
  - If stays ~900 → PC-A's NIC is the saturation point (normal)
  - If jumps to ~1800 → switch has headroom (great)

### Latency
- [ ] From PC-A: `ping -c 100 <PC-B-IP>`
  - avg: **______ ms** stddev: **______ ms**
- [ ] Target: sub-1 ms avg, low stddev. Higher = switch congestion or Wi-Fi contamination somewhere.

### Topology reconnaissance
- [ ] Ask lab admin (or check yourself): one big switch for the lab? Per-table switches uplinked to a core? Managed or unmanaged?
- [ ] Sketch the switch topology: ___________________________________

### Red flags
- [ ] **Throughput <500 Mbit/s** — cable, NIC, or switch problem. Big-guy tier at risk.
- [ ] **Ping stddev >5ms** — congestion. Pipeline parallelism will suffer badly.
- [ ] **Multi-subnet routing** — flag to prof, may need VLAN reconfiguration.

---

## Track C — Single-node inference baseline (60-90 min)

Do this on the **worst** of your 3 PCs (worst-case-first is more informative). Reuse the same PC for all sub-tests.

### Install (pick one)
- [ ] `ollama` (easier): `curl -fsSL https://ollama.com/install.sh | sh`
- [ ] **OR** `llama.cpp` (more control): clone + `cmake -B build && cmake --build build --config Release -j`

### Model downloads (~5 GB total)
- [ ] Qwen2.5-Coder-3B-Instruct GGUF Q4_K_M (~2 GB) — primary candidate
- [ ] Qwen2.5-Coder-1.5B GGUF Q4 (~900 MB) — fallback if 3B is too slow
- [ ] Qwen2.5-Coder-7B GGUF Q4_K_M (~4.4 GB) — stretch test; expect it to barely fit

### Benchmark prompt (use the SAME prompt for every run for comparability)
```
Write a Python function that reverses a singly linked list.
Include a docstring, type hints, and 2 doctests.
```

### Measurement matrix — fill this in

| Model | Config | TTFT (s) | Tok/s | RAM used | GPU used | Notes |
|---|---|---|---|---|---|---|
| Qwen 1.5B | CPU only | | | | | |
| Qwen 1.5B | + GPU offload | | | | | |
| Qwen 3B | CPU only | | | | | |
| Qwen 3B | + GPU offload | | | | | |
| Qwen 7B | CPU only | | | | | |
| Qwen 7B | + GPU offload | | | | | |

- Time to first token (TTFT): time from submit until first token appears
- Tok/s: total_output_tokens ÷ generation_time
- llama.cpp offload flag: `-ngl 20` (adjust based on VRAM)
- ollama: automatic offload; check with `ollama ps`

### Verdict
- [ ] User-node model choice: ______________________
- [ ] Reasoning: ______________________________________

---

## Track D — Distributed sanity check (60 min) — *only if A-C look OK*

This is the make-or-break test for the whole big-guy tier. If it fails today, plan pivots to prima.cpp on your next lab visit.

### Setup
- [ ] Build llama.cpp with RPC support on 2 PCs: `cmake -B build -DGGML_RPC=ON`
- [ ] On PC-A: `./build/bin/rpc-server -p 50052`
- [ ] On PC-B: run llama.cpp main with RPC backend pointing at PC-A:
  ```
  ./build/bin/llama-cli -m qwen-7b-q4.gguf --rpc <PC-A-IP>:50052 -ngl 999 -p "..."
  ```

### Measure
- [ ] Distributed tok/s (7B split across A+B): **______**
- [ ] Compare to single-PC 7B CPU-only from Track C: **______**
- [ ] Ratio: distributed/single = **______** (expect <1 — that's the point)

### Failure test
- [ ] Kill PC-A's rpc-server mid-generation. PC-B client behavior:
  - [ ] Clean error message?
  - [ ] Hang?
  - [ ] Crash?
- Notes: ______________________________________

### Verdict
- [ ] Distributed inference works on this hardware: **YES / NO**
- [ ] If NO — root cause: ______________________________________

---

## Track E — Cloud API pool setup (30 min, anywhere with internet)

Can be done at home the night before if you want.

Sign up and grab an API key for each:
- [ ] **Groq** — https://console.groq.com — key: `gsk_...`
- [ ] **Google AI Studio** (Gemini) — https://aistudio.google.com — key: `AIza...`
- [ ] **OpenRouter** — https://openrouter.ai — key: `sk-or-...`
- [ ] **Cerebras** — https://cloud.cerebras.ai — key: `csk-...`
- [ ] **Mistral La Plateforme** — https://console.mistral.ai — key: `...`

- [ ] Save all keys to a password manager. **NEVER commit to a repo.**

For each, run one test call (use `curl` or a short Python script):
- [ ] Groq — works? Rate limit noted: ____ req/min, ____ tok/min
- [ ] Google — works? Rate limit noted: ____ req/min, ____ req/day
- [ ] OpenRouter — works? Rate limit noted: ____ req/min, ____ req/day
- [ ] Cerebras — works? Rate limit noted: ____
- [ ] Mistral — works? Rate limit noted: ____

---

## End-of-day summary — fill this out before leaving the lab

> Our lab PCs actually have: **_________________** GPU, **____ GB** RAM, **_______** OS, admin: **YES / NO**.

> Network node-to-node measures **____ Mbit/s**, latency **____ ms**.

> Best small model for user nodes is **_________________**, delivering **____ tok/s** on our worst PC.

> Distributed inference across 2 PCs: **WORKS / DOESN'T WORK / NEEDS PRIMA.CPP**.

> We have **____ / 5** cloud API keys working.

> **Biggest blocker discovered:** ______________________________________

> **Next visit priority:** ______________________________________

---

## Bonus if you finish early

- [ ] `git clone https://github.com/Lizonghang/prima.cpp` on one PC, read README, try to build.
- [ ] Run `nmap -sP <your-subnet>/24` to count how many PCs are actually alive on the LAN right now.
- [ ] Time a full 3-PC boot from cold start — you'll need this figure for the fleet-out phase.
- [ ] Photograph the physical layout, note which PCs are on which tables, sketch the cable runs (helps decide column-per-cluster feasibility).

---

## Things to intentionally NOT do today

- Don't try OpenClaw. Known-weak security model on shared lab hardware = bad time.
- Don't try to install anything on all 40 PCs. Today is measurement, not deployment.
- Don't fine-tune or train anything. Out of scope for the whole project.
- Don't wire up master/user-node/big-guy end-to-end. That's Phases 3-6, weeks away.
