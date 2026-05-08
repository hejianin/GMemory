# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup

```bash
conda create -n GMemory python=3.12
conda activate GMemory
pip install -r requirements.txt
```

Copy `template.env` to `.env` and fill in your OpenAI-compatible API credentials:
```
OPENAI_API_BASE=<Your URL>
OPENAI_API_KEY=<Your API Key>
```

## Running Experiments

**Shell script (edit `run_mas.sh` to configure options):**
```bash
./run_mas.sh
```

**Direct Python invocation:**
```bash
python tasks/run.py --task alfworld --reasoning io --mas_memory g-memory --max_trials 30 --mas_type autogen --model <model_name>
python tasks/run.py --task pddl     --reasoning io --mas_memory g-memory --max_trials 30 --mas_type autogen --model <model_name>
python tasks/run.py --task fever    --reasoning io --mas_memory g-memory --mas_trials 15 --mas_type autogen --model <model_name>
python tasks/run.py --task sciworld --reasoning io --mas_memory g-memory --max_trials 30 --mas_type autogen --model <model_name>
```

**Key CLI arguments:**
- `--task`: `alfworld` | `fever` | `pddl` | `sciworld`
- `--mas_type`: `autogen` | `dylan` | `macnet`
- `--mas_memory`: `empty` | `chatdev` | `metagpt` | `voyager` | `generative` | `memorybank` | `g-memory`
- `--reasoning`: `io` (only option currently)
- `--successful_topk`, `--failed_topk`, `--insights_topk`: retrieval counts from memory
- `--threshold`: similarity threshold for trajectory retrieval
- `--hop`: graph traversal hops in TaskLayer
- `--use_projector`: enable role-specific insight projection

Results and memory databases are written to `.db/<model>/<task>/<mas_type>/<memory_type>/`.

## Architecture

The codebase separates the core MAS framework (`mas/`) from task-specific implementations (`tasks/`).

### `mas/` — Core Framework

- **`mas/llm.py`**: `GPTChat` wraps the OpenAI SDK with retry logic. Reads `OPENAI_API_BASE`/`OPENAI_API_KEY` from env at import time. LLM config (temperature, max_token) comes from `configs/configs.yaml`.
- **`mas/agents/base.py`**: `Agent` holds a name, role, system instruction, and a `ReasoningBase` module. `agent.response()` prepends the system instruction to every call.
- **`mas/mas.py`**: `MetaMAS` is the abstract base for all MAS workflows — it manages an `agents_team` dict and delegates `build_system()` / `schedule()` to subclasses.
- **`mas/reasoning/`**: `ReasoningBase` + `ReasoningIO` (direct input→output reasoning). New reasoning strategies go here.
- **`mas/memory/common.py`**: Core data structures:
  - `AgentMessage` — a single agent turn (system instruction, user instruction, response)
  - `StateChain` — a list of `nx.DiGraph` states, each representing one env step's message graph
  - `MASMessage` — wraps a full task: description, trajectory string, `StateChain`, and label (success/failure)
- **`mas/memory/mas_memory/memory_base.py`**: `MASMemoryBase` — abstract base with `add_memory()` and `retrieve_memory()` interface.
- **`mas/module_map.py`**: Maps string names to classes — the single place to register new reasoning or memory modules.

### G-Memory (`mas/memory/mas_memory/GMemory.py`)

Three-tier hierarchical memory:
1. **Interaction Graph** (inside `StateChain`): Per-step message graphs tracking which agent said what to whom.
2. **TaskLayer**: A `networkx.Graph` of task nodes connected by similarity (threshold 0.7). Persisted as `task_layer_graph.pkl`. Used for k-hop neighborhood expansion during retrieval. Also supports FINCH clustering for the merge step.
3. **InsightsManager**: A scored list of rules persisted as `insights.json`. Rules are added/edited/removed via LLM-generated operations (`ADD`, `EDIT`, `REMOVE`, `AGREE`). Finetuning triggers every `rounds_per_insights` tasks; merging triggers every 20 tasks.

Memory flow: `add_memory()` → sparsify trajectory (remove negative-reward steps) → add to ChromaDB + TaskLayer → conditionally finetune/merge insights. `retrieve_memory()` → k-hop graph expansion → rerank by LLM importance score → return top-k successes, failures, and insights.

### `tasks/` — Task-Specific Code

- **`tasks/run.py`**: Entry point. Builds `TaskManager` (env + recorder + MAS workflow), then calls `build_mas()` and `run_task()` in sequence.
- **`tasks/envs/`**: One `BaseEnv` subclass per task. `set_env()` returns `(task_main, task_description)`; `step()` returns `(observation, reward, done)`.
- **`tasks/mas_workflow/`**: One `MetaMAS` subclass per MAS type (autogen, dylan, macnet). Implements `build_system()` and `schedule()`.
- **`tasks/prompts/`**: Per-task system prompts and few-shot examples. `get_dataset_system_prompt()` and `get_task_few_shots()` are the public API.
- **`tasks/configs.yaml`**: Per-task env config paths, `max_steps`, `few_shots_num`, and MAS hyperparameters.
- **`configs/configs.yaml`**: Global LLM settings (temperature, max_token) and embedding model.

### Data

Datasets must be downloaded separately and placed in `data/`:
- ALFWorld: `data/alfworld/alfworld_tasks_suffix.json`
- PDDL: `data/pddl/test.jsonl`
- FEVER: `data/fever/fever_dev.jsonl` (only first 100 entries used)
- SciWorld: `data/sciworld/test.jsonl`
