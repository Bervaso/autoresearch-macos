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

Starting val_bpb: **1.285372** (best from sessions 1 and 2).
Target val_bpb: **below 1.200**.
This is a 6.6% improvement — ambitious but achievable through architectural
innovation and smarter search. Hyperparameter tweaking alone will not get
you there. You need ideas from recent ML research.

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

## RESEARCH DIRECTIONS — WHERE TO LOOK FOR BREAKTHROUGHS

The previous sessions exhausted basic hyperparameter tuning. To reach 1.200
you need architectural ideas. Here are directions worth exploring, drawn
from recent ML research (you know these from your training data):

### High priority — most likely to help on MPS with fixed 5-min budget:

**1. Efficient attention alternatives**
- Linear attention (e.g. RWKV-style or RetNet-style recurrence) — O(T)
  instead of O(T²). On MPS where attention is slow, this could allow
  more training steps.
- GLA (Gated Linear Attention) — combines gating with linear complexity.
  Implementable in pure PyTorch, no new dependencies.

**2. Better MLP design**
- The current ReLU² activation is good but explore GLU variants that don't
  require SwiGLU's extra parameters (e.g. bilinear layers, gated ReLU).
- Mixture of depths: not all tokens need the same computation.

**3. Improved residual and normalization**
- Pre-norm vs post-norm vs sandwich-norm — try combinations.
- Deep-norm initialization (scale residuals by N^(-1/4)) for more stable
  training with fewer steps.
- RMSNorm with learnable scale already exists — try removing the scale
  entirely (plain RMSNorm) to reduce parameters and speed up steps.

**4. Smarter learning rate schedules**
- Trapezoidal schedule (warmup → flat → linear decay) instead of cosine.
- Cyclic LR within the 5-minute budget — multiple mini-cycles may explore
  the loss landscape better than one long warmdown.
- WSD (Warmup-Stable-Decay) schedule recently shown to outperform cosine.

**5. Token mixing alternatives**
- The current window pattern SSSL uses local attention. Try combining with
  a single global pooling layer (like in Hyena or H3) instead of full
  attention for global context.

**6. Optimizer improvements**
- Adan optimizer: uses both first and second-order gradient differences.
  Implementable from scratch in ~30 lines of PyTorch.
- SOAP: Shampoo-style preconditioner, shown to outperform AdamW on small
  models. Also implementable without new dependencies.

### Medium priority — worth trying if high-priority ideas stall:

- Rotary embedding base frequency tuning (YaRN-style scaling for short seqs)
- Stochastic depth (randomly skip layers during training)
- Weight sharing across layers (ALBERT-style) — fewer params, more steps
- Learned token merging (reduce sequence length mid-network)

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

Branch: `autoresearch/mar20-v3`

LOOP FOREVER:

1. Check git state.
2. Choose an experiment — prefer architectural ideas from the research
   directions above over hyperparameter tweaks.
3. Modify `train.py`, git commit.
4. Run: `uv run train.py > run.log 2>&1 &` then `sleep 340 && grep "^val_bpb:\|^peak_vram_mb:" run.log`
5. Log to `results.tsv`.
6. Keep if improved, `git reset --hard HEAD~1` if not.
7. Every 10 experiments: review and optionally update `program.md`.

**Timeout**: Kill and discard any run exceeding 10 minutes.
**Crashes**: Fix trivial bugs and retry once. Skip fundamentally broken ideas.
**NEVER STOP**: Run until the human interrupts you. Do not ask permission to continue.
