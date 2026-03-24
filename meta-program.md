# meta-program.md — Session 3 Research Constitution

This file governs session 3. It has two layers: an immutable constitution
(rules you must NEVER change) and an evolvable strategy (which you ARE
allowed to update as you learn). Your research loop modifies both `train.py`
AND `program.md` — but never this file.

---

## THE CONSTITUTION (NEVER MODIFY THESE RULES)

These rules are permanent. No experiment result, no matter how promising,
justifies changing them:

1. **Never modify `prepare.py`** — the evaluation harness and time budget are fixed.
2. **Never add dependencies** — only what is already in `pyproject.toml`.
3. **`val_bpb` is the only metric** — do not invent alternative metrics.
4. **5-minute time budget is sacred** — do not change `TIME_BUDGET` in any way.
5. **`results.tsv` format is fixed** — always: `commit / val_bpb / memory_gb / status / description`.
6. **Never modify `meta-program.md`** — this file is read-only.
7. **Never stop the loop** — run until manually interrupted by the human.

---

## YOUR MISSION

Starting val_bpb: **1.261259** (best from session 3).
Target val_bpb: **below 1.200**.
This is a 4.8% improvement. Hyperparameter tweaking is exhausted.
Only fundamental architectural changes can get you there.
The agent in session 3 identified **Mixture of Experts** as the most
promising unexplored direction. Start there.

---

## HOW THIS SESSION WORKS

You have two levers, not one:

### Lever 1 — Modify `train.py` (as before)
Same as previous sessions. Every experiment commits a change, runs for 5
minutes, keeps or discards.

### Lever 2 — Modify `program.md` (NEW)
After every 10 experiments, stop and review `program.md`. Ask yourself:
- Is my current search strategy working?
- Am I exploring the right directions?
- Should I add new heuristics, paper ideas, or escape strategies?

If yes, update `program.md` with what you've learned, then git commit the
change with message: `program.md: [what you changed and why]`

This makes your research strategy itself an evolving artifact.

---

## RESEARCH DIRECTIONS FOR SESSION 4

Sessions 1-3 have exhausted hyperparameter tuning and basic architecture
search. The current best architecture is:
- 4 layers: 3 causal depthwise conv (kernel=15) + 1 attention
- 256 dim, 2 heads, MLP 8x ReLU², LayerNorm
- Value embeddings on attention layer only
- Sequence curriculum 512→2048, full bf16
- ~2395 steps in 5 minutes on MPS

### Priority 1 — Mixture of Experts (start here)

MoE gives 2x model capacity at the same compute cost. Implementation:
- Add a router (small linear layer) that scores each token for 2 expert MLPs
- Each token is processed by only 1 expert (top-1 routing)
- Net effect: same FLOPs per step, but 2x the MLP parameters
- Add a small load balancing loss to prevent all tokens routing to one expert
- Try on the MLP layers of the conv blocks first (cheaper than attention)

This is the single most promising untried direction. Spend at least
5 experiments here before moving on.

### Priority 2 — Learned token merging (if MoE stalls)

Reduce sequence length mid-network using learned pooling:
- After the first 2 conv layers, merge adjacent token pairs (T → T/2)
- Process the merged sequence through the remaining layers
- This halves the cost of the attention layer → more training steps
- Implementable in pure PyTorch with learned linear projection for merging

### Priority 3 — Gated Linear Attention (if token merging stalls)

Replace the single attention layer with Gated Linear Attention (GLA):
- O(T) complexity instead of O(T²) — much faster on MPS
- Retains gating mechanism for selectivity
- Implementable from scratch in ~40 lines of PyTorch, no new dependencies
- Reference: GLA paper (2024) — the core idea is: h_t = G_t * h_{t-1} + k_t^T v_t

### Priority 4 — WSD learning rate schedule (quick win attempt)

Warmup-Stable-Decay: replace current cosine warmdown with:
- 0-5%: linear warmup
- 5-80%: constant LR
- 80-100%: linear decay to 0
Recent work shows WSD often outperforms cosine at small scale.

### What NOT to try (already exhausted in sessions 1-3):
- Depth changes (too slow on MPS)
- SwiGLU, GELU activations (ReLU² is best here)
- Batch size changes (2^14 is optimal)
- Most LR/schedule variations
- Standard self-attention in place of conv (too slow)

---

## ESCAPING LOCAL OPTIMA (carry forward from session 2)

If 5+ consecutive experiments are discarded, you are stuck. Use these
strategies in order:

1. **Near-miss retry** — retry the closest discard on the current config.
2. **Combination shot** — combine two discarded ideas that might interact.
3. **Deliberate step back** — accept up to 0.010 worse once to enter a new
   architectural region. Tag as `[exploratory]`.
4. **Radical architectural jump** — replace a core component entirely
   (e.g. swap self-attention for linear attention in all layers at once).

Always state your hypothesis before making a change. Random flailing wastes
experiments.

---

## EXPERIMENT LOOP

Branch: `autoresearch/mar20-v4`

LOOP FOREVER:

1. Check git state.
2. Choose an experiment — follow the priority order in Research Directions.
3. Modify `train.py`, git commit.
4. Run: `uv run train.py > run.log 2>&1 &` then `sleep 340 && grep "^val_bpb:\|^peak_vram_mb:" run.log`
5. Log to `results.tsv`.
6. Keep if improved, `git reset --hard HEAD~1` if not.
7. Every 10 experiments: review and optionally update `program.md`.

**Timeout**: Kill and discard any run exceeding 10 minutes.
**Crashes**: Fix trivial bugs and retry once. Skip fundamentally broken ideas.

**NEVER STOP**: Run until the human interrupts you. Do not ask permission
to continue. Do not summarize and stop. Do not say you are "done". Even if
you feel you have exhausted all ideas, think harder — combine previous
near-misses, try more radical changes, revisit the research directions above.
The loop runs until the human types a message interrupting you. Period.

**Context window management**: Every 20 experiments, write a brief status
update to `program.md` under a new heading (e.g. "## Session 4 exps 1-20").
This ensures your learnings are preserved even if the session restarts.
Do this as part of the loop — it takes 30 seconds and saves everything.
