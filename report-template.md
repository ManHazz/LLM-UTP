# <SESSION TITLE> — Session <N> Report

**Date:** <YYYY-MM-DD>
**Author:** <name>
**Session number:** <n>
**Objective:** <one-sentence goal of this session — what question you set out to answer or what capability you set out to build>

---

## 1. Executive Summary

<3-6 sentences. Written LAST, after the rest of the report. State: what you did, the headline result (with the key number), the main takeaway, and any blocker or surprise. A reader who only reads this section should understand the session's contribution.>

Key numbers:
- <metric 1>: <value>
- <metric 2>: <value>

---

## 2. Setup & Changes from Previous Session

<What changed vs the last session. Keep it small — full hardware inventory only if it changed. Otherwise reference the previous report and just note diffs.>

### Nodes participating
| Role | Hostname | IP | Notes (if changed) |
|---|---|---|---|
| Driver | | | |
| Worker | | | |

### Software / config changes
- <e.g. added N new nodes, upgraded llama.cpp to commit X, downloaded new model Y>

---

## 3. Pre-flight Issues Encountered

<Problems that surfaced before the actual experiment could run. Kernel mismatches, missing packages, network hiccups, etc. Include root cause + resolution. Skip this section entirely if nothing broke.>

### <issue 1 title>
- **Symptom:**
- **Root cause:**
- **Resolution:**

---

## 4. Methodology

<How you ran the experiment. Enough detail that someone else could reproduce it. Specify: model(s), prompt(s), llama-cli/llama-bench flags, how many runs per condition, how you averaged.>

### Test parameters
- **Model(s):**
- **Prompt(s):**
- **Runs per cell:**
- **Flags used:**

### Commands

```bash
# Worker startup:

# Driver invocation:
```

---

## 5. Results

<Raw numbers. Table first, then chart if applicable. No interpretation yet — save that for section 6.>

### <table title>
| <col1> | <col2> | <col3> |
|---|---:|---:|

### <chart title, if any>

```
<ASCII chart or reference to a saved image>
```

---

## 6. Analysis

<Interpretation of the numbers. Why they came out that way. What theory/prior work predicts vs what you observed. Discrepancies and possible causes.>

### <observation 1 heading>

<Explanation.>

### <observation 2 heading>

<Explanation.>

---

## 7. Reproducibility Appendix

<Copy-pasteable command blocks in the order needed to redo this session on a clean cluster. Should assume prior sessions' setup already done unless this session changed something.>

### <step 1>

```bash
```

### <step 2>

```bash
```

---

## 8. Next Steps

<Concrete follow-ups for the next session. Prioritized. Include both "obvious extension of what worked" and "investigate the thing that didn't work.">

1.
2.
3.

---

## 9. References

<Papers, docs, blog posts, and other primary sources. Cite anything you extrapolated from or that provided methodology. Use full URLs — future you may not remember the paper name.>

-
-
-

---

<!--
Notes for filling in this template:

- Section 1 (Executive Summary) is written LAST. Write everything else first, then distill the top.
- Section 2 keeps the report readable — no need to re-list identical hardware every time. Just diffs.
- Section 3 only exists if something broke. Delete this section entirely if the session was clean.
- Section 5 is raw numbers only. No sentences like "as expected, tok/s increased." Save that for section 6.
- Section 6 is where you can speculate, argue, and connect to theory. But mark speculation as speculation.
- Section 7 should be flat command blocks, not narrative. Someone in a hurry should be able to copy-paste sequentially.
- Section 8 becomes section 4's methodology for the next session's report.
- Delete this HTML comment block when finalizing.
-->
