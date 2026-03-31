# Aurora Swarm — Communication Patterns and Use Cases

## Architecture

A Python async master process communicates with 1000–4000 GPT-OSS-120B
agent instances running on Aurora compute nodes. Each agent exposes an
HTTP endpoint (host:port). The master reads a hostfile written by the
Aurora startup script, opens a pooled connection to every endpoint, and
orchestrates work using the patterns described below.

All patterns share the same core machinery: an `AgentPool` backed by
`asyncio` + `aiohttp` with semaphore-throttled concurrency (default 512
in-flight requests) and a shared TCP connection pool (default 1024
connections). Agents can be tagged in the hostfile (e.g. `role=critic`,
`node=aurora-0042`) and selected at runtime with `pool.by_tag()`,
`pool.sample()`, or `pool.select()`.


## Pattern 1: Broadcast

**What it does:** Sends the identical prompt to every agent in the pool
and collects all responses.

**Two variants:**

- `broadcast(pool, prompt)` — fire-and-gather, returns all responses.
- `broadcast_and_reduce(pool, prompt, reduce_prompt)` — two-phase:
  first broadcast to all agents, then feed the collected responses to a
  single "reducer" agent that produces a final synthesis.

**When to use it:**

- Ensemble classification — ask 4000 agents the same yes/no question,
  aggregate by majority vote to get high-confidence answers.
- Redundant generation — produce thousands of candidate answers to the
  same question, then filter by quality.
- Parallel scoring — every agent evaluates the same candidate (molecule,
  policy, proof) from a different angle.

**Aggregation:** `majority_vote()` for categorical outputs,
`statistics()` for numeric outputs, `best_of()` or `top_k()` to select
the highest-quality responses.

**Scaling note:** With 4000 agents and a 512-concurrency semaphore, a
broadcast completes in ~8 waves. The bottleneck is the slowest agent in
each wave.


## Pattern 2: Scatter-Gather

**What it does:** Distributes different work items across agents — each
agent gets a unique prompt. Results are gathered back in input order.

**Two variants:**

- `scatter_gather(pool, prompts)` — send `prompts[i]` to `agent[i]`.
- `map_gather(pool, items, prompt_template)` — higher-level: takes a
  list of work items and a template with an `{item}` placeholder,
  formats each item into a prompt, then scatters.

**When to use it:**

- Compound library screening — evaluate 4000 molecules, one per agent.
- Parameter space search — each agent tests a different hyperparameter
  configuration or simulation condition.
- Dataset processing — split a large corpus into chunks, each agent
  processes one chunk (extract entities, summarize, classify).
- Monte Carlo sampling — each agent runs one sample with different
  random seeds.

**Aggregation:** `structured_merge()` when agents return JSON,
`statistics()` for numeric results, `failure_report()` to track which
items failed.

**Scaling note:** Perfectly parallel. If you have more items than agents,
agents process multiple items in round-robin. Throughput scales linearly
with pool size.


## Pattern 3: Hierarchical Tree-Reduce

**What it does:** Organizes agents into a tree structure. Leaf agents
produce initial responses. Groups of responses (default 50 per group)
are fed to "supervisor" agents that summarize them. Supervisors'
summaries are recursively grouped and summarized until a single final
answer remains.

**Parameters:**

- `prompt` — the leaf-level task (with `{item}` placeholder if scattering).
- `reduce_prompt` — the summarization task (with `{responses}` and
  `{level}` placeholders).
- `fanin` — how many responses each supervisor handles (default 50).
- With 4000 agents and fanin=50: level 1 produces 80 summaries, level 2
  produces 2 summaries, level 3 produces the final answer. Three
  reduction rounds total.

**When to use it:**

- Literature synthesis — each agent reads one paper abstract, tree
  produces a unified review across thousands of papers.
- Multi-hypothesis debate — agents propose diverse hypotheses,
  supervisors identify consensus and contradictions at each level.
- Large-scale ensemble reasoning — agents answer independently,
  supervisors distill the best reasoning chains.
- Survey analysis — each agent analyzes one respondent or data source,
  tree aggregates themes and statistics.

**Key advantage over broadcast+reduce:** The aggregation work is
distributed across agents rather than bottlenecked on a single reducer.
Each supervisor only needs to read ~50 responses, not 4000. The
supervisors are themselves GPT-OSS-120B instances, so they perform
intelligent summarization rather than naive concatenation.

**Scaling note:** Depth grows as log(N)/log(fanin). With 4000 agents and
fanin=50, depth is 3. Total agent-calls = N + N/fanin + N/fanin^2 + ...
which is roughly N * (1 + 1/fanin), so overhead is ~2% beyond the leaf
computation.


## Pattern 4: Blackboard (Shared-State Swarm)

**What it does:** Agents collaborate through a shared mutable workspace
divided into named sections (e.g. "hypotheses", "critiques",
"synthesis"). The session runs in rounds. Each round:

1. Every agent reads the current board state.
2. A role-specific prompt function generates a customized prompt for
   each agent based on its role and the board contents.
3. All agents respond in parallel.
4. Responses are written back to the board under the agent's role section.
5. A convergence check determines whether to continue.

**Agent roles** are assigned via hostfile tags (`role=hypotheses`,
`role=critiques`, `role=synthesis`) and can be any string. The prompt
function uses the role to tailor behavior:

- **Proposers** generate new hypotheses or refine existing ones.
- **Critics** identify weaknesses, missing evidence, or logical gaps.
- **Synthesizers** summarize the debate and identify strongest positions.

**When to use it:**

- Open-ended scientific discovery — explore a problem space without
  a predetermined decomposition.
- Adversarial verification — proposer agents make claims, critic agents
  attack them, forcing hypotheses to improve over rounds.
- Iterative refinement toward consensus — agents converge on an answer
  through multi-round debate.
- Red-team / blue-team analysis — one group tries to find flaws in a
  design while another defends it.

**Convergence criteria:** User-defined function. Examples:
- Fixed number of rounds.
- Synthesizer outputs stop changing significantly.
- All critics report "no remaining objections."
- A quality score exceeds a threshold.

**Scaling note:** Each round is a full broadcast, so cost is
O(rounds * N). Typically 5–10 rounds with a subset of the pool
(e.g. 200 agents) rather than the full 4000. Best combined with
sub-pool selection for tractability.


## Pattern 5: Pipeline (Multi-Stage DAG)

**What it does:** Defines a sequence of stages, each served by a pool of
agents. The output of one stage flows as input to the next. Each stage
can use a different number of agents and different prompt templates.

**Stage definition:**

- `name` — human-readable label.
- `prompt_template` — must contain `{input}` placeholder.
- `n_agents` — how many agents this stage uses (can vary per stage).
- `output_transform` — optional function to reshape outputs before
  feeding to the next stage.
- `output_filter` — optional function to drop low-quality outputs.

**Execution modes:**

- `reuse_agents=True` — all stages draw from the same pool (agents
  handle different roles at different times).
- `reuse_agents=False` — pool is partitioned, each stage gets a
  dedicated subset (no contention).

**When to use it:**

- Scientific method workflow:
  - Stage 1: Hypothesis generation (1000 agents)
  - Stage 2: Experiment design (1000 agents)
  - Stage 3: Statistical analysis (500 agents)
  - Stage 4: Final synthesis (100 agents)
- Multi-step reasoning: draft → critique → revise → evaluate.
- Data processing: extract → transform → validate → summarize.
- Progressive filtering: generate 4000 candidates → score → keep top
  100 → refine → keep top 10 → final ranking.

**Fan-out/fan-in variant:** `fan_out_fan_in()` is a convenience for the
common two-stage case: broadcast a question to N workers, then send all
responses to a single collector for synthesis.

**Scaling note:** Total wall-clock time is the sum of stage times (stages
are sequential). Within each stage, work is parallel. A 4-stage pipeline
with 1000 agents per stage takes ~4x the time of a single broadcast but
produces much more refined output.


## Combining Patterns

These patterns are composable. Practical workflows often combine them:

- **Tree inside a pipeline stage:** Stage 2 of a pipeline uses
  tree-reduce to synthesize the outputs of stage 1 before feeding
  to stage 3.
- **Blackboard with broadcast seeding:** Broadcast a question to all
  agents first, then run a blackboard session among a curated subset
  to refine the best answers.
- **Scatter + tree:** Scatter 4000 different items, then tree-reduce
  the results into a summary. This is what `tree_reduce(items=...)`
  does natively.
- **Pipeline with blackboard stage:** One stage of the pipeline is a
  blackboard session rather than a simple scatter.


## Aggregation Strategies

| Aggregator          | Input                  | Output                  | Best for                    |
|---------------------|------------------------|-------------------------|-----------------------------|
| `majority_vote`     | Categorical responses  | Winner + confidence     | Classification ensembles    |
| `concat`            | Any text               | Joined string           | Collecting diverse opinions |
| `best_of`           | Scored responses       | Single best             | Quality selection           |
| `top_k`             | Scored responses       | Top k responses         | Candidate shortlisting      |
| `structured_merge`  | JSON responses         | Merged list + errors    | Structured data collection  |
| `statistics`        | Numeric values         | mean/std/median/min/max | Numeric ensemble estimates  |
| `failure_report`    | Any batch              | Success/fail counts     | Monitoring and debugging    |


## Practical Considerations

**Concurrency control:** The semaphore (default 512) prevents
overwhelming the network or the agents. Tune based on Aurora's
interconnect bandwidth and agent response times.

**Fault tolerance:** Agents may fail or timeout. All patterns handle
this gracefully — failed responses are flagged with `success=False` and
an error message. Aggregators skip failed responses by default.
`failure_report()` provides a diagnostic summary.

**Hostfile tags:** The startup script can tag agents with metadata
(node, GPU index, role, specialization). This enables runtime
selection without code changes: `pool.by_tag("role", "critic")`.

**Checkpointing:** For long-running sessions (especially blackboard),
the `Blackboard.snapshot()` method returns a serializable dict that
can be saved to disk and reloaded.
