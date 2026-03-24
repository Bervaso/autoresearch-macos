# autoresearch

This is an experiment to have the LLM do its own research.

## Setup

To set up a new experiment, work with the user to:

1. **Agree on a run tag**: propose a tag based on today's date (e.g. `mar5`). The branch `autoresearch/<tag>` must not already exist — this is a fresh run.
2. **Create the branch**: `git checkout -b autoresearch/<tag>` from current master.
3. **Read the in-scope files**: The repo is small. Read these files for full context:
   - `README.md` — repository context.
   - `prepare.py` — fixed constants, data prep, tokenizer, dataloader, evaluation. Do not modify.
   - `train.py` — the file you modify. Model architecture, optimizer, training loop.
4. **Verify data exists**: Check that `~/.cache/autoresearch/` contains data shards and a tokenizer. If not, tell the human to run `uv run prepare.py`.
5. **Initialize results.tsv**: Create `results.tsv` with just the header row. The baseline will be recorded after the first run.
6. **Confirm and go**: Confirm setup looks good.

Once you get confirmation, kick off the experimentation.

## Experimentation

Each experiment runs on a single GPU. The training script runs for a **fixed time budget of 5 minutes** (wall clock training time, excluding startup/compilation). You launch it simply as: `uv run train.py`.

**What you CAN do:**
- Modify `train.py` — this is the only file you edit. Everything is fair game: model architecture, optimizer, hyperparameters, training loop, batch size, model size, etc.

**What you CANNOT do:**
- Modify `prepare.py`. It is read-only. It contains the fixed evaluation, data loading, tokenizer, and training constants (time budget, sequence length, etc).
- Install new packages or add dependencies. You can only use what's already in `pyproject.toml`.
- Modify the evaluation harness. The `evaluate_bpb` function in `prepare.py` is the ground truth metric.

**The goal is simple: get the lowest val_bpb.** Since the time budget is fixed, you don't need to worry about training time — it's always 5 minutes. Everything is fair game: change the architecture, the optimizer, the hyperparameters, the batch size, the model size. The only constraint is that the code runs without crashing and finishes within the time budget.

**VRAM** is a soft constraint. Some increase is acceptable for meaningful val_bpb gains, but it should not blow up dramatically.

**Simplicity criterion**: All else being equal, simpler is better. A small improvement that adds ugly complexity is not worth it. Conversely, removing something and getting equal or better results is a great outcome — that's a simplification win. When evaluating whether to keep a change, weigh the complexity cost against the improvement magnitude. A 0.001 val_bpb improvement that adds 20 lines of hacky code? Probably not worth it. A 0.001 val_bpb improvement from deleting code? Definitely keep. An improvement of ~0 but much simpler code? Keep.

**The first run**: Your very first run should always be to establish the baseline, so you will run the training script as is.

## Output format

Once the script finishes it prints a summary like this:

```
---
val_bpb:          0.997900
training_seconds: 300.1
total_seconds:    325.9
peak_vram_mb:     45060.2
mfu_percent:      39.80
total_tokens_M:   499.6
num_steps:        953
num_params_M:     50.3
depth:            8
```

Note that the script is configured to always stop after 5 minutes, so depending on the computing platform of this computer the numbers might look different. You can extract the key metric from the log file:

```
grep "^val_bpb:" run.log
```

## Logging results

When an experiment is done, log it to `results.tsv` (tab-separated, NOT comma-separated — commas break in descriptions).

The TSV has a header row and 5 columns:

```
commit	val_bpb	memory_gb	status	description
```

1. git commit hash (short, 7 chars)
2. val_bpb achieved (e.g. 1.234567) — use 0.000000 for crashes
3. peak memory in GB, round to .1f (e.g. 12.3 — divide peak_vram_mb by 1024) — use 0.0 for crashes
4. status: `keep`, `discard`, or `crash`
5. short text description of what this experiment tried

Example:

```
commit	val_bpb	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	switch to GeLU activation
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
```

## The experiment loop

The experiment runs on a dedicated branch (e.g. `autoresearch/mar5` or `autoresearch/mar5-gpu0`).

LOOP FOREVER:

1. Look at the git state: the current branch/commit we're on
2. Tune `train.py` with an experimental idea by directly hacking the code.
3. git commit
4. Run the experiment: `uv run train.py > run.log 2>&1` (redirect everything — do NOT use tee or let output flood your context)
5. Read out the results: `grep "^val_bpb:\|^peak_vram_mb:" run.log`
6. If the grep output is empty, the run crashed. Run `tail -n 50 run.log` to read the Python stack trace and attempt a fix. If you can't get things to work after more than a few attempts, give up.
7. Record the results in the tsv
8. If val_bpb improved (lower), you "advance" the branch, keeping the git commit
9. If val_bpb is equal or worse, you git reset back to where you started

The idea is that you are a completely autonomous researcher trying things out. If they work, keep. If they don't, discard. And you're advancing the branch so that you can iterate. If you feel like you're getting stuck in some way, you can rewind but you should probably do this very very sparingly (if ever).

**Timeout**: Each experiment should take ~5 minutes total (+ a few seconds for startup and eval overhead). If a run exceeds 10 minutes, kill it and treat it as a failure (discard and revert).

**Crashes**: If a run crashes (OOM, or a bug, or etc.), use your judgment: If it's something dumb and easy to fix (e.g. a typo, a missing import), fix it and re-run. If the idea itself is fundamentally broken, just skip it, log "crash" as the status in the tsv, and move on.

## Escaping local optima

Pure greedy hill-climbing (keep only improvements) risks getting permanently stuck in a local minimum. You must actively detect and escape this.

**Detect a plateau**: If the last 5 consecutive experiments were all `discard`, you are likely stuck in a local minimum. Acknowledge this explicitly before choosing your next move.

**When stuck, reason over the full history**: Read `results.tsv` carefully. Ask yourself:
- Which discarded experiments were close to the best (within 0.005 val_bpb)? These are near-misses worth revisiting with the current optimized config — the context has changed since they were tried.
- Which ideas were tried in isolation but never combined? Two individually neutral changes might interact positively.
- Which directions haven't been explored at all? Look for gaps: have you tried changing the normalization strategy, the residual connection structure, the tokenizer sequence packing, or the LR schedule shape?

**Controlled exploration moves**: When stuck, choose one of these strategies deliberately:

1. **Near-miss retry**: Pick the best discarded experiment (lowest val_bpb among discards) and retry it on the current best config — conditions may have changed enough to make it work now.
2. **Combination shot**: Combine two previously discarded ideas that seem theoretically complementary. Commit both changes together and treat it as one experiment.
3. **Deliberate step back**: Accept a result up to 0.010 worse than the current best *once* if it opens a genuinely new architectural direction you haven't explored. Track this in the description as `[exploratory]`. If the next experiment from this new starting point doesn't improve things, revert all the way back to the previous best.
4. **Radical jump**: Make a large multi-parameter change (e.g. simultaneously change architecture + optimizer settings). Greedy search can't reach configurations that require coordinated changes — this can.

**Record your reasoning**: When using an escape strategy, note it in the description field of results.tsv so the history stays interpretable. Example: `[near-miss retry] SwiGLU on current batch/LR config`.

**Stay scientific**: Exploration is not random flailing. Every move should have a hypothesis. If you can't articulate why a change might help, don't make it.

**NEVER STOP**: Once the experiment loop has begun (after the initial setup), do NOT pause to ask the human if you should continue. Do NOT ask "should I keep going?" or "is this a good stopping point?". The human might be asleep, or gone from a computer and expects you to continue working *indefinitely* until you are manually stopped. You are autonomous. If you run out of ideas, think harder — read papers referenced in the code, re-read the in-scope files for new angles, try combining previous near-misses, try more radical architectural changes. The loop runs until the human interrupts you, period.

As an example use case, a user might leave you running while they sleep. If each experiment takes you ~5 minutes then you can run approx 12/hour, for a total of about 100 over the duration of the average human sleep. The user then wakes up to experimental results, all completed by you while they slept!

## Session 3 Learnings (experiments 1-10)

### Key breakthrough: Conv mixer replaces attention in local layers
- Replacing 3/4 attention layers with causal depthwise conv (kernel=15) gave -0.018 bpb
- On MPS, attention is the bottleneck. Conv is 2.3x faster → 2270 vs 1000 steps in 5 min
- The speed gain from more training steps outweighs the quality loss from weaker local mixing
- One full attention layer (the last/global layer) still provides long-range context

### What didn't work:
- SwiGLU activation (1.305 vs 1.285) — ReLU² is better at this small scale
- Increasing DEPTH without conv (1.338 with DEPTH=6) — too slow, undertrained
- HEAD_DIM 128→64 (1.318) — fewer dims per head hurt
- MLP 8x (1.289) — near miss but slightly worse
- Removing value embeddings (1.308) — VE are essential for quality
- Smaller batch size (1.302) — 16K batch already optimal
- Warmup + reduced warmdown (1.296) — current schedule is well-tuned
- DEPTH=8 conv model (1.274) — too slow despite cheap conv layers
- DEPTH=6 conv model (1.285) — too slow, 3 VE layers expensive

### Session 3 Learnings (experiments 11-30)

**Improvements kept**: MLP 8x (1.266→1.265), bf16 (1.265), curriculum 512→2048 (1.265), global ctx (1.264)
**Total session 3 improvement**: 1.285 → 1.265 (-1.6%)

**What didn't work (exps 11-30)**:
- MLP 10x: too slow (1.267)
- Conv kernel 31: too slow (1.273)
- Conv kernel 7: too little context (1.271)
- DEPTH 5-8: too slow (1.269-1.285)
- Wider model 384 dim: too slow (1.266)
- Multi-scale conv (3+15): slower, no gain (1.268)
- Higher MATRIX_LR: diverges (1.267)
- Embedding dropout: model is undertrained not overfitting (1.272)
- Z-loss: interferes with training (1.275)
- Weight tying: conflicting LR needs (crash)
- Strided attention: catastrophic quality loss (1.404)
- 3-phase curriculum 256→1024→2048: too aggressive (1.268)

### Current architecture (best: 1.264608)
- 4 layers: 3 conv (kernel=15 + cumulative mean global ctx) + 1 attention
- 256 dim, 2 heads (HEAD_DIM=128), MLP 8x ReLU²
- Value embeddings on attention layer only (2.1M params)
- Full bf16, sequence curriculum (512 first half, 2048 second half)
- ~2179 steps in 5 min on MPS

### Guiding principles:
1. **Speed is king on MPS** — more steps > better architecture
2. **Value embeddings are essential** for quality
3. **DEPTH=4, MLP 8x** is the sweet spot for speed/quality
4. **Model is undertrained (36M tokens, 12M params)** — don't regularize
5. **Hyperparams are near-optimal** — LR/schedule tuning gives < 0.002
6. **Need fundamentally different approach** to reach 1.200 (5% more)

### Experiments 36-40 (schedule/norm/LR tuning): all discarded
- WARMDOWN_RATIO 0.4: 1.267, WARMDOWN_RATIO 0.6: 1.265 — 0.5 is optimal
- FINAL_LR_FRAC 0.0: 1.268 — 0.1 is optimal
- Sandwich norm: 1.269 — slower, no gain
- UNEMBEDDING_LR 0.008: 1.272 — too high
- Also failed: stochastic depth (1.278), all-conv no attention (1.436), deep supervision (1.283)

### Assessment after 40 experiments:
The model is at a strong local optimum. All hyperparameters are near-optimal. Incremental changes consistently fail. To reach 1.200 (another 5.2%):
- Need a fundamentally different architecture class
- Or a different way to use the 5 min budget
- MoE remains the most promising unexplored direction

## Session 4 exps 1-10: all discarded

### MoE (exps 1-3): FAILED on MPS
- Top-1 hard routing: MPS bf16 doesn't support scatter/gather in backward
- Both-compute MoE: 2x cost for no benefit (1.275)
- Channel gating: extra linear too expensive (1.269)
- Split-proj (6x shared fc, 2 projs): dual proj slower than single 8x (1.272)
**Conclusion**: MoE is architecturally incompatible with MPS speed constraints. Any extra computation per step is wasted.

### Schedule (exps 4-5): FAILED
- WSD 5/75/20: warmup still hurts (1.270)
- WSD 0/70/30 to 0: current schedule is better (1.267)

### Token merging (exp 6): CATASTROPHIC
- T→T/2 after layer 1: 86ms/step (very fast!) but train/eval mismatch → 3.551 val_bpb
- Eval needs full T, model only trained at T/2

### Linear attention (exp 7): FAILED
- Chunk-64 with KV state: cross-chunk linear attention too weak (1.361)

### Speed-focused (exps 8-10): FAILED
- MLP 6x no curriculum: more steps but less capacity (1.264)
- Heterogeneous conv:6x attn:12x: attn layer too slow (1.263)
- Inverse conv:10x attn:4x: conv MLP too slow (1.267)

### Key insight after 10 experiments:
The baseline architecture (3 conv + 1 attn, 8x MLP, curriculum) is remarkably well-optimized.
The 1.261 optimum is incredibly hard to escape. Every change makes things worse.

## Session 4 exps 11-40

### Breakthroughs:
- **exp11: SLSL pattern** (2conv+2attn) → 1.260 (LayerNorm made 2-attn viable)
- **exp19-20: curriculum 60→70/30** → 1.259 (more short-seq steps helps)
- **exp25: 384 dim + MLP 4x** → 1.256 (wider model with SLSL is better!)
- **exp30: MLP 5x at 384 dim** → 1.255 (sweet spot between 4x and 6x)

### Current best architecture (1.255415):
- **384 dim, 3 heads** (HEAD_DIM=128)
- **SLSL** pattern (2 conv + 2 attn, alternating)
- **MLP 5x** with ReLU², LayerNorm, bf16
- **70/30 curriculum** (512/2048), VE on both attn layers
- ~1664 steps in 5 min on MPS

### Confirmed optimal (exps 31-40):
- WARMDOWN 0.5 is optimal (0.4 and 0.6 both worse)
- 70/30 curriculum is optimal (80/20 too little full-length)
- SLSL > SSLL > SLLL > LLLL (alternating is key)
- HEAD_DIM=128 (64 still bad)
- Full MHA (GQA 1KV worse)
- DEPTH 4 > DEPTH 3 > DEPTH 5 (at 384 dim)
- MATRIX_LR 0.02 still optimal
- BATCH 16K still optimal

### What to try next:
- Different dim: try 320 dim (2.5*128) with MLP 6x
- conv kernel tuning at 384 dim (currently 15)
- Different curriculum reshape factor (try 2x instead of 4x)
