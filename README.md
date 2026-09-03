# GLM-5.3-Flash on Dell Pro Max with GB10 ×2 — Full Record of a No-Go Verdict (Failure Case Study)

> We spent a full day deploying and testing GLM-5.3-Flash on the dual-machine setup and ultimately ruled **no generation upgrade**.
> A failure case is worth as much as a success for pitfall avoidance: this book covers what we hit, how we diagnosed it, and why "it boots" is not the same as "it's usable".
> (Note: the no-go verdict applies to "running these weights locally on two machines"; cloud API is unaffected by these local stability issues — but it has a separate output-leak risk, see the quality verdict section.)

## Hardware and the Stacks We Tried

| Item | Spec |
|---|---|
| Machines | Dell Pro Max with GB10 ×2 (sm_121, 128GB unified memory each) |
| Stack A | vLLM TP2 dual-machine (same shape as the DSV4F book in this series) |
| Stack B | EXL3 (exllamav3) dual-machine recipe (`glm-5.3-flash-exl3-2x-spark`) |
| Verdict | **No-go: not for production**. The concurrent baseline (DeepSeek V4 Flash Vision Exp, dual-machine) was more stable across the board |

## What We Hit (in chronological order)

1. **Unstable from init**: the vLLM stack (same image line as the DSV4F book in this series) reached serving state in only 1 of 5 cold starts; the rest wedged during weight loading/compilation. The operator paths for this model on sm_121 are not mature enough.
2. **Deterministic crash at 120K long context**: same class as the 95K death line in this series' Flash-Next book — long context is advertised, but on this stack it crashes deterministically past a length threshold. Confirmed again: **the safe long-context line must be measured per model, per stack**.
3. **Single-machine load wedge (#109 class)**: loading the large weights blows through unified memory and the machine hangs half-dead. The breakthrough combo (**only needed for single-machine loading, and it's a high-risk system tweak — you must roll it back when done**):
   ```bash
   sudo fallocate -l 48G /swap-tmp && sudo mkswap /swap-tmp && sudo swapon /swap-tmp
   sudo sysctl vm.watermark_scale_factor=200
   # Roll back after loading completes:
   sudo swapoff /swap-tmp && sudo rm /swap-tmp && sudo sysctl vm.watermark_scale_factor=10
   ```
   Plus lowering the GPU memory utilization ratio; **dual-machine TP2 sharded loading does not need any of this** (each machine only takes half the weights).
4. **Serving parameters wiped by the install script**: tuned parameters vanished after re-running install — you must write them directly into the env file's `${VAR:=}` default-value lines (same as pitfall #6 in this series' DSV4F book; we paid the tuition on this model first).
5. **Image digest-pin breaks on offline transfer**: `docker save | load` drops the repo digest (same as pitfall #9 in the DSV4F book, also learned in this round).

## Quality Verdict (why "it runs" still gets a no-go)

In the windows where it did run, we completed a six-dimension evaluation (code generation / debugging / Chinese / long-form logic / agentic / math, each on a 0-100 in-house scale): **it won only one dimension, code generation (41.5, with the baseline lower on that dimension)**; it trailed or tied everywhere else. And in a production context, the stability record (init 1/5, 120K crash) is a one-vote veto.
The cloud API version of the same weights carries a separate risk worth knowing: in 2026-09, calling it through a third-party API aggregation gateway, we repeatedly observed the thinking stream leaking into the content field in English (whether it reproduces depends on gateway and server versions — verify yourself: check whether content contains `<think>` or blocks of English reasoning). If you use models from this series anywhere for user-facing output, put a budget clamp on thinking and check content for `<think>` residue.

## Reusable Takeaways from This Book

- **The bar for "deployed successfully" is not a 200 from the service — it's passing all three gates: init success rate + safe long-context line + steady-state concurrency**. Our three-gate process: cold start ×5 and count the success rate → measured length ladder (method in this series' Flash-Next book) → steady-state concurrency (6-way mix: 2 short Q&A + 2 code generation + 2 medium-long documents, 30 minutes, pass = zero worker crashes and aggregate throughput not trending to zero).
- Before putting a new model on two machines, scout with a small quantized version on one machine first; only go TP2 once the operator paths look basically mature — TP2's failure surface is the square of a single machine's.
- A no-go verdict doesn't mean deleting the archive: keep the weights and configs in place (storage is cheap); retesting after an engine iteration costs far less than re-downloading and reconfiguring.

---
*RyanAI Lab · All numbers measured on our resident environment. Updated 2026-09. Issues welcome.*
