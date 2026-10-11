# LLM-UTP

deploying a local LLM cluster across ~40 PCs in a lab.

no GPU servers, no cloud budget — just the hardware already in the room, distributed inference, and a cloud API fallback pool for when local isn't enough.

---

> 📖 **full documentation, architecture, and lab session logs → [Wiki](../../wiki)**

---

## quick overview

| tier | role |
|------|------|
| master node | routing, load balancing, API gateway |
| user nodes | per-user inference (small models, most lab PCs) |
| big-guy tier | heavy models split across GPU machines |

models: Qwen2.5-Coder family (1.5B / 3B / 7B), served via `ollama` or `llama.cpp` RPC.

---

**maintained by** [@ManHazz](https://github.com/ManHazz) · UTP Computer Engineering
