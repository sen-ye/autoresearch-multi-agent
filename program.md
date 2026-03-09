# Multi-Agent Autoresearch Swarm

This is a competitive and cooperative multi-agent experiment where multiple LLM subagents conduct independent machine learning research, share insights, and compete to achieve the lowest `val_bpb`.

## System Architecture & Roles

The system consists of one **Master Agent** and **N Subagents** (where N is the total number of subagents specified by the user at launch).

### 1. Master Agent Initialization

If you are the Master Agent, your role is strictly to orchestrate the environment:

1. Receive the target number of subagents (**N**) from the user.
2. Create a `worktree` folder in the current root directory.
3. Inside `worktree`, create **N** independent subdirectories named `agent_0` through `agent_{N-1}`.
4. Copy the base repository files (e.g., `README.md`, `prepare.py`, `train.py`, `pyproject.toml`, `program.md`, `results.tsv`, `lessons.md`, `insights.md`) into each of the **N** subdirectories.
5. Spawn the **N** Subagents. Provide **exactly** this prompt to each subagent: "You are Sub Agent {i}. You can only use GPU {i} (CUDA_VISIBLE_DEVICES={i}). The uv env is at xxx . There are {N} total agents in this swarm. Read program.md in your directory to understand your rules, constraints, and goals. Begin your experiment loop."

## Experimentation Setup (Subagent)

Before you begin your loop, ensure your workspace is ready:

1. Read the in-scope files in your directory for context (`README.md`, `prepare.py`, `train.py`).
2. Do not modify `prepare.py`. It is read-only and contains the fixed evaluation, data loading, tokenizer, and constants.
3. Verify data exists in `~/.cache/autoresearch/`.
4. Initialize your local `results.tsv` with just the header row.
5. Create empty `lessons.md` and `insights.md` files in your directory.

## Rules of Engagement

You run on a single GPU. The training script runs for a fixed time budget of 5 minutes.

**What you CAN do:**

* Modify your local `train.py` — this is the only code file you edit. Everything is fair game: model architecture, optimizer, hyperparameters, training loop, batch size, model size, etc.

**What you CANNOT do:**

* Modify `prepare.py`.
* Modify another agent's files.
* Install new packages or add dependencies. Use what's in `pyproject.toml`.
* Modify the evaluation harness (`evaluate_bpb` in `prepare.py`).

**Evaluation:**
The goal is simple: get the lowest `val_bpb`. Since the time budget is fixed (5 minutes), you don't need to worry about training time — it's always 5 minutes. VRAM is a soft constraint; some increase is acceptable for meaningful gains, but do not OOM.

**Simplicity criterion:**
All else being equal, simpler is better. A 0.001 val_bpb improvement that adds 20 lines of hacky code is not worth it. A 0.001 val_bpb improvement from deleting code is a huge win.

## Logging & Knowledge Sharing

You are required to maintain three distinct logs in your directory to facilitate the swarm's collective intelligence:

### 1. `results.tsv`

Log every experiment here (tab-separated). The headers are:
`commit` \t `val_bpb` \t `memory_gb` \t `status` \t `description`
*(Use 0.000000 for crashes. Status must be `keep`, `discard`, or `crash`)*

### 2. `lessons.md`

A running log of empirical facts. What worked? What crashed? What blew up VRAM? Keep this brief and factual so other agents can parse it quickly. Example:

* *Attempted GeLU instead of ReLU: Worse val_bpb (+0.05).*
* *Doubled batch size to 64: OOM crash.*
* *Added gradient clipping (1.0): Stable, slight improvement (-0.002).*

### 3. `insights.md`

A higher-level log for your architectural hypotheses. Why do you think a certain mechanism failed? What is the overarching direction you are exploring? Read peers' insights to branch out into uncharted territory instead of overlapping.

## The Experiment Loop

As Agent `i`, you operate autonomously in `worktree/agent_i/`.

**Initial Step: The Baseline**

* **If you are Agent 0:** Your very first task is to establish the baseline. Do NOT modify `train.py` on your first attempt. Execute the script as-is, evaluate it, and log the initial `val_bpb` in your `results.tsv` (status: `keep`, description: `baseline`) and `lessons.md`. Once the baseline is logged, enter the loop below.
* **If you are Agent 1 to N-1:** Skip the baseline. You are free to begin radical exploration and modify `train.py` immediately.

**LOOP FOREVER:**

1. **Scout:** Read the `lessons.md` and `insights.md` of the other agents (maybe empty initially). Incorporate their successes and avoid their failures.
2. **Hypothesize:** Formulate an experimental idea and write your theory in your local `insights.md`.
3. **Execute:** Modify `train.py` directly. Git commit your changes.
4. **Run:** Execute `CUDA_VISIBLE_DEVICES=i uv run train.py > logs/run_exp_agent{i}.log 2>&1`. (Do NOT let output flood your context).
5. **Evaluate:** Extract metrics using `grep "^val_bpb:\|^peak_vram_mb:" logs/run_exp_agent{i}.log`.
* If the run crashed (empty grep), read the stack trace with `tail -n 50 logs/run_exp_agent{i}.log`. Fix minor bugs. If fundamentally broken, discard.

6. **Log:** * Record the result in `results.tsv`.
* Document what happened in `lessons.md`.

7. **Branch Management:**
* If `val_bpb` improved (lower), keep the commit and advance.
* If `val_bpb` is equal or worse, `git reset --hard HEAD~1` to revert to your last good state.

**NEVER STOP:** Do NOT pause to ask the human if you should continue. The human expects you to work indefinitely. If you run out of ideas, read your peers' files, read the code again, combine near-misses, or try radical simplifications. You are an autonomous researcher in a competitive swarm. May the best agent win.