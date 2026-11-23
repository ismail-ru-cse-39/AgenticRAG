# A Survey of Multi-Step and Complex Query Answering: Architectural Evolution, Optimization, and Cost–Quality Trade-offs

## Abstract

Multi-step and complex query answering requires a system to identify missing
information, retrieve evidence over several turns, integrate heterogeneous
sources, and construct a justified answer. Retrieval-augmented generation
(RAG) began as a relatively static pipeline, but it has evolved into adaptive,
corrective, graph-based, and agentic systems that make retrieval and tool-use
decisions while reasoning. This survey organizes that evolution by
architectural coupling and control flow. It distinguishes one-shot, modular,
iterative, corrective, structure-aware, and agentic approaches; discusses
query decomposition, interleaved reasoning, tool use, memory, reflection, and
multi-agent coordination; and analyzes retrieval, generation, serving, and
orchestration optimizations. We examine benchmark and evaluation practices,
including multi-hop QA datasets and evidence, faithfulness, relevance,
latency, token use, and failure analysis. We argue that production architecture
selection should be modeled as a constrained optimization problem rather than
as a search for the highest benchmark score. The main direction of future work
is to reduce unnecessary retrieval and reasoning while retaining enough
adaptivity for hard queries, with explicit error budgets and verifiable
evidence.

## 1. Introduction

A complex question is not necessarily long. It is one for which a complete
answer depends on dependencies across evidence: finding a first entity, using
its attributes to identify a second, comparing two sources, or combining
information that changes over time. Examples include bridge questions
("Where was the director of X born?"), comparison questions, temporal and
aggregate questions, document-level synthesis, and questions that require
external tools.

Large language models (LLMs) can solve some such tasks from parametric
knowledge, but they do not expose the provenance of that knowledge and may
hallucinate when the needed facts are absent or conflicting. RAG provides
external evidence, while agentic systems add a control loop that can decide
what to retrieve, which tool to call, whether evidence is sufficient, and how
to revise the plan.

We use **complexity** to mean a demand on computation or evidence that cannot
be satisfied reliably by one retrieval and one generation call. It may arise
from:

1. **hop depth**: information needed in one step determines the next query;
2. **reasoning type**: comparison, aggregation, constraint satisfaction,
   temporal reasoning, or negation;
3. **corpus structure**: facts distributed across passages or relationships;
4. **source heterogeneity**: tables, APIs, documents, graphs, or live data;
5. **uncertainty**: contradictory or incomplete evidence; and
6. **open-endedness**: broad synthesis without a small set of known facts.

## 2. Problem Formulation

Given query $q$, corpus or tool set $C$, and response budget $B$, a system
produces answer $a$ and evidence $E$:

$$
\max_{a,\pi,E}\; Q(a,E\mid q)
\quad\text{subject to}\quad
C_{\text{token}}+C_{\text{compute}}+C_{\text{tool}}\le B
$$

Here $\pi$ is a retrieval and reasoning policy. Quality $Q$ may combine answer
correctness, citation support, coverage, calibration, and user utility. This
formulation exposes the central issue in agentic RAG: quality is a function of
both the final response and the policy that produced it.

A simple stateful process is:

```text
state = {question, plan, evidence, intermediate_answers}
while evidence is insufficient and budget remains:
    action = choose_action(state)
    observation = execute(action)
    state = update(state, observation)
return synthesize(state)
```

Actions include direct answering, query rewriting, one or more retrievals,
reranking, document reading, tool calls, reasoning, verification, and
termination.

## 3. Taxonomy

| Class | Control | Typical mechanism | Strength | Limitation |
|---|---|---|---|---|
| Naive RAG | Fixed | Retrieve top-$k$, generate once | Simple, low latency | Poor on dependencies and ambiguity |
| Modular RAG | Fixed modules | Rewrite, retrieve, rerank, generate | Tunable components | Modules may optimize local objectives |
| Iterative RAG | Query-conditioned loop | Decompose or interleave retrieval and reasoning | Better evidence recall | Cost grows with steps |
| Corrective RAG | Feedback-driven | Assess evidence, refine or replace sources | Reduces bad context | Correction detector may fail |
| Structure-aware RAG | Corpus structure | Graphs, tables, summaries, communities | Better global and relational access | Indexing is expensive and lossy |
| Agentic RAG | State-dependent policy | Reflection, planning, tools, memory | Flexible and extensible | Harder to control, evaluate, and secure |

### 3.1 Naive RAG

Naive RAG combines dense or sparse retrieval with one generator call. Lewis et
al. established RAG as a retrieval-augmented conditional generation
architecture [1]. This paradigm is effective for fact lookup, but a query
is only retrieved once and no component verifies whether the returned context
is sufficient.

Sparse retrieval is strong for exact names and identifiers. Dense retrieval
handles paraphrases and semantic similarity. Hybrid retrieval combines them,
often by reciprocal rank fusion or learned reranking. Reranking at later stages
is usually more accurate than adding retrieval depth uniformly, because
candidate recall and final context relevance have different objectives.

### 3.2 Modular RAG

Modular systems separate query analysis, rewriting, retrieval, reranking,
context compression, generation, and citation construction. This separation
allows each stage to use specialized models or indexes and makes debugging
easier.

However, local metrics may not predict end-to-end answer quality. A module that
maximizes passage recall can introduce noise; a compression module that
minimizes tokens can remove the evidence needed for a later inference step.
End-to-end measurement is therefore preferred for consequential tuning.

### 3.3 Iterative and Interleaved RAG

Iterative RAG uses the previous result to form a new query. A common pattern is
query decomposition: split a question into subquestions, solve each one, and
aggregate the results. It is effective for explicit conjunctions and bridge
questions, but decomposition itself can introduce an irreversible error.

IRCoT interleaves chain-of-thought reasoning with retrieval [2]. Each reasoning
step retrieves evidence for the next step, which can improve recall for
multi-hop queries. The cost is repeated prompting and repeated retrieval.
FLARE retrieves when a generated response lacks sufficient confidence or an
upcoming span requires support [3].

Adaptive-RAG further routes a query to no retrieval, single-step RAG, or
iterative RAG according to estimated complexity [4]. The routing policy is
valuable because not every query benefits from the expensive path.

### 3.4 Corrective RAG

Corrective RAG evaluates retrieved evidence before or after generation.
CRAG grades document relevance and can trigger local correction or web
retrieval when confidence is low [5]. Self-RAG trains a language model to emit
reflection tokens that decide whether to retrieve and whether a passage
supports a generated segment [6].

These designs improve on "retrieve everything and hope" behavior, but they
replace a fixed pipeline with learned or heuristic stopping decisions. They
require calibration: over-retrieval raises cost, while under-retrieval creates
ungrounded answers.

### 3.5 Structure-Aware RAG

GraphRAG extracts entities and relationships, identifies hierarchical
communities, and summarizes their contents [7]. It can be useful for global
queries such as themes spanning many documents, though it incurs substantial
index construction cost and may discard fine-grained evidence.

Knowledge-graph systems additionally support typed relations and constraint
paths. In some benchmark settings, apparently complex graph questions reduce to
simpler link prediction or one-hop problems [8]. Therefore, an architecture
benchmark should verify that evaluation questions genuinely require the
represented hops.

### 3.6 Agentic RAG

Agentic RAG adds a policy loop around retrieval and generation. ReAct
alternates reasoning traces and external actions [9], while tool-learning
systems such as Toolformer learn when to invoke APIs [10]. A practical agent
typically separates the following concerns:

- **planner**: proposes or revises a plan;
- **executor**: calls tools or retrieval services;
- **critic**: verifies support and detects contradictions;
- **memory**: retains short-term observations or durable facts;
- **controller**: applies budgets, permissions, retry limits, and stopping rules.

Reflection can occur within a model, as in Self-RAG, or across a multi-agent
workflow. Multi-agent designs can isolate retrieval, critique, and answer
synthesis, but they multiply latency and create unproductive disagreement unless
the controller owns convergence and stopping.

## 4. Architectural Evolution

The evolution is not merely from "RAG" to "agent." The independent axes are
control, evidence source, reasoning, and structure:

```text
Static RAG
  |  fixed retrieval -> retrieval with rewriting/reranking
  v
Modular RAG
  |  local components -> feedback from evidence
  v
Adaptive / Iterative RAG
  |  fixed stages -> stateful tools and policy
  v
Agentic RAG
  |  single agent -> controlled tool and multi-agent workflows
  v
Reliable autonomous systems
     explicit budgets, provenance, verification, and governance
```

Each transition trades stronger expressiveness for additional failure modes.
Static RAG has few moving parts, but weak adaptability. Iterative RAG can
follow dependencies, but may over-retrieve or loop. Agents can integrate APIs
and memories, but may act on stale or untrusted retrieved content.

The preferred mature architecture is not maximally agentic. It is a
minimal-control system that meets the quality and risk constraints of the
target workload.

## 5. Optimization

### 5.1 Retrieval Optimization

Retrieval optimization can improve both quality and cost:

- tune chunk size and overlap for the question distribution;
- use hybrid sparse/dense retrieval;
- retrieve candidates broadly, then rerank narrowly;
- compress or summarize passages after relevance filtering;
- use query rewriting, expansion, or decomposition only when justified;
- cache embeddings and repeated retrieval results;
- add metadata and time filters; and
- stop after a marginal-gain threshold rather than a fixed depth.

Index quality matters as much as model choice. Approximate nearest-neighbor
indexes such as HNSW reduce search cost but introduce recall/configuration
trade-offs. Production retrieval should report recall against a validation
set, not only total latency.

### 5.2 Query Routing

Routing sends easy queries to direct generation, medium queries to standard
RAG, and hard queries to iterative or agentic paths. A good router uses query
features or calibrated confidence and is trained against end-to-end utility.
It should expose a quality floor: a low-cost route must not be selected
whenever the model estimates a consequential factual risk.

Routing reduces average cost but may be worse than a fixed method on a
particular slice. Workload-sensitive evaluation should report quality and cost
distributions, not only averages.

### 5.3 Generation and Context Optimization

Context construction is the major interface between retrieval and generation.
Helpful techniques include deduplication, diversity-aware selection,
cross-document coreference, evidence compression, citation-aware prompting,
and position-aware ordering.

Long contexts are not automatically beneficial. They can dilute useful evidence
and increase prefill cost. Compact evidence packs should prefer independently
verifiable passages and remove redundant context.

### 5.4 Serving Optimization

Standard serving optimizations remain important for RAG:

- KV caching and paged KV caches;
- Flash Attention and memory-efficient attention;
- continuous batching;
- prefix caching;
- speculative decoding;
- quantization;
- smaller models for routing and retrieval judgments; and
- distributed retrieval and result caching.

These optimizations differ in where they save cost. KV and attention
optimization reduce memory or decode cost, batching improves throughput, and
speculative decoding lowers per-request latency. They should be measured under
real concurrency, because a technique that helps a single request can degrade
tail throughput.

### 5.5 Agent and Orchestration Optimization

Agent cost is dominated by repeated model calls and prompt growth. The
controller should use deadlines and token limits, avoid unnecessary reflection,
cache observations, merge repeated tool calls, and permit no more than a small
number of retries.

For tool-heavy tasks, schema size is itself an optimization problem. A
large tool catalog should be retrieved and filtered before tool selection,
rather than supplied in every prompt.

## 6. Cost–Quality Trade-offs

The cost of RAG is multidimensional:

| Dimension | Primary source | Cheap policy | Quality risk |
|---|---|---:|---|
| Compute | Model calls, attention | Direct generation or small models | Missed evidence or weak reasoning |
| Tokens | Prompts and completions | Short context, no reflection | Lost provenance or evidence |
| Retrieval | ANN search and reranking | Few passages and no reranker | Lower recall |
| Tool calls | APIs, search, databases | Cache or reuse observations | Stale or unsupported results |
| Latency | serial steps | Parallel retrieval and generation | Waiting on correctness checks |
| Indexing | graph/entity extraction | Vector-only corpus | Weak relational/global synthesis |

A cost-aware policy can optimize a utility function:

$$
U(a,q)=\hat Q(a,q)-\lambda_C C-\lambda_L L-\lambda_R R
$$

where $C$, $L$, and $R$ represent token cost, latency, and operational risk.
The coefficients encode deployment priorities. One application may require
high $\lambda_R$ for medical or legal questions, while another can tolerate
shorter citations for low-stakes exploratory search.

### 6.1 When Extra Retrieval Helps

Additional retrieval is likely to help when:

- the question has explicit dependencies;
- query terms are ambiguous;
- one result has low confidence;
- evidence is spread across documents;
- the corpus changes frequently; or
- citation or contradiction analysis is required.

### 6.2 When Extra Retrieval Hurts

Additional retrieval is less useful when:

- the answer is already supported by a trusted source;
- the corpus has little semantic diversity;
- the model has sufficient parametric knowledge;
- additional passages are redundant; or
- the system is already near a recall ceiling.

The useful Pareto frontier is often a set of routing policies, not one method.
A fixed-heavy policy may provide higher quality, while a fixed-light policy
reduces cost. A query router can occupy the region between them and should be
evaluated against both baselines.

### 6.3 Reliability and Security Costs

Agentic systems have additional costs for validation, permissions, isolation,
and rollback. Retrieved text can contain prompt-injection instructions;
therefore, content is untrusted data, not a valid authority for tool behavior.
Production controllers must treat tool and memory outputs as observations,
constrain side effects, and require authorization for external actions.

## 7. Evaluation

### 7.1 Datasets

| Dataset | Core task | Important limitation |
|---|---|---|
| HotpotQA | Explainable multi-hop QA | Can contain shortcut artifacts |
| 2WikiMultihopQA | Two-document and compositional QA | Structured distributions differ from production |
| MuSiQue | Connected reasoning across hops | Harder, but still answer-focused |
| NQ / TriviaQA | Fact and passage QA | Often do not require multi-hop retrieval |
| ASQA | Ambiguous, multi-answer QA | Requires answer coverage and attribution |
| Long-document benchmarks | Cross-document/section synthesis | Metrics depend heavily on chunking |

An evaluation should separate retrieval quality from final-answer quality.
The same answer can be correct for unsupported reasons, and the same
retrieved evidence can support different generated answers.

### 7.2 Metrics

Useful measures include:

- exact match and token/semantic answer quality;
- recall and precision at $k$ for evidence;
- supporting-passage ranking and citation coverage;
- faithfulness or entailment of claims;
- answer relevance;
- grounding and provenance completeness;
- calibration and abstention;
- tool success and invalid-action rate;
- median, p95, and p99 latency;
- input, cache-read, and output tokens;
- number of retrieval and tool calls;
- cost per successful answer, not cost per request; and
- recovery and failure budgets.

RAGAS and ARES provide frameworks for automated answer/relevance and
faithfulness evaluation [11,12]. They do not eliminate human evaluation:
LLM judges can share biases with the generation model, and citation metrics
must be checked against the actual source segments.

### 7.3 Experimental Protocol

A defensible study should:

1. hold corpus, retrieval, and generation models constant while changing
   architecture;
2. report number of attempts and model sizes;
3. include no-retrieval, fixed-RAG, and fixed-iteration baselines;
4. stratify results by question complexity;
5. separate successful from abstained answers;
6. report p95/p99 latency and token distributions;
7. test duplicate, contradictory, temporal, and adversarial evidence;
8. include cost-quality Pareto curves;
9. use held-out routing data; and
10. repeat experiments with confidence intervals.

## 8. Failure Modes

| Failure | Symptom | Mitigation |
|---|---|---|
| Missed hop | Retrieval lacks the entity discovered later | Iterative or interleaved retrieval |
| Early decomposition | Subquestions cannot be combined | Joint planner or final consistency check |
| Retriever noise | Irrelevant passages crowd evidence | Reranking and evidence filtering |
| Context poisoning | Retrieved text changes tool or answer behavior | Source isolation and instruction hierarchy |
| Hallucinated provenance | Fluent citation without support | Claim-level entailment and citation validation |
| Contradiction | Sources disagree | Temporal/source weighting and conflict reporting |
| Retrieval loop | Agent repeats without progress | State de-duplication, budgets, and forced termination |
| Routing error | Hard query takes cheap route | Quality floor and calibrated uncertainty |
| Stale evidence | Corpus and current event disagree | Freshness metadata and temporal retrieval |
| Cascading tool call | One invalid action causes later changes | Simulate, authorize, and validate side effects |

The most important mitigations are operational rather than purely model-level:
budget caps, tool allowlists, audit logs, evidence provenance, user-visible
uncertainty, and safe fallbacks.

## 9. Open Challenges

### 9.1 Adaptive Computation

Future systems need uncertainty-aware policies that allocate retrieval depth,
model size, and reasoning steps per query. Training an agent only for final
answer accuracy may encourage excessive tool use; a multi-objective reward
must penalize unnecessary cost and unsafe actions.

### 9.2 Verified and Evidence-Aware Generation

Claim-level citation verification should become a standard stage rather than a
post-hoc promise. The verifier must distinguish an answer that is supported,
supported with an omitted caveat, contradicted, or simply not in the corpus.

### 9.3 Dynamic and Multi-Source Knowledge

Systems must merge document, graph, database, and API evidence while tracking
freshness and authority. Conflict resolution should return uncertainty rather
than silently choosing a source.

### 9.4 Global Queries

Global synthesis remains expensive. Dynamic graph traversal and learned
abstraction can improve comprehensiveness, but the best split between offline
index construction and query-time work remains workload-dependent.

### 9.5 Evaluation of Agents

Benchmarks need to evaluate intermediate decisions, evidence efficiency,
tool-call validity, recovery, and cost. Static datasets are also vulnerable to
contamination. Live, auditable, and periodically renewed benchmarks should
measure deployment conditions.

### 9.6 Privacy and Governance

Complex agents can combine private data with public tools. Policies are needed
for data minimization, retrieval authorization, memory retention, and
cross-user isolation. A prompt is not a security boundary.

## 10. Practical Recommendations

1. Start with a strong hybrid retrieval baseline and a reranker.
2. Measure retrieval recall, answer correctness, support, latency, and cost
   separately.
3. Add decomposition or interleaved retrieval only for complex slices.
4. Route easy queries to direct or short-RAG paths.
5. Enforce evidence sufficiency and stop conditions.
6. Keep context compact and deduplicated.
7. Use cache, batching, and prefix reuse before expensive model scaling.
8. Constrain tools and require confirmation for consequential actions.
9. Report Pareto curves and p95/p99 behavior.
10. Preserve citation and source provenance from retrieval through final output.

The main architectural lesson is that adaptive systems do not need to perform
the maximum amount of work for every query. They need to allocate work
according to evidence need and risk.

## References

[1] P. Lewis, E. Perez, A. Piktus, F. Petroni, V. Karpukhin, N. Goyal,
H. Küttler, M. Lewis, W.-t. Yih, T. Rocktäschel, S. Riedel, and D. Kiela,
"Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks,"
*Advances in Neural Information Processing Systems*, 2020.
https://arxiv.org/abs/2005.11401

[2] H. Trivedi, N. Balasubramanian, T. Khot, and A. Sabharwal,
"Interleaving Retrieval with Chain-of-Thought Reasoning for Knowledge-Intensive
Multi-Step Questions," 2022.
https://arxiv.org/abs/2212.10509

[3] Z. Jiang, F. F. Xu, L. Gao, Z. Sun, Q. Liu, J. Dwivedi-Yu, Y. Yang,
J. Callan, and G. Neubig, "Active Retrieval Augmented Generation,"
*EMNLP*, 2023. https://arxiv.org/abs/2305.06983

[4] S. Jeong, J. Baek, S. Cho, and S. J. Hwang, "Adaptive-RAG: Learning to
Adapt Retrieval-Augmented Large Language Models through Question Complexity,"
*NAACL*, 2024. https://arxiv.org/abs/2403.14403

[5] S. Yan, J. Gu, Y. Zhu, and D.-Y. Zeng, "Corrective Retrieval Augmented
Generation," 2024. https://arxiv.org/abs/2401.15884

[6] A. Asai, Z. Wu, Y. Wang, A. Sil, and H. Hajishirzi, "Self-RAG: Learning to
Retrieve, Generate, and Critique through Self-Reflection," *ICLR*, 2024.
https://arxiv.org/abs/2310.11511

[7] D. Edge, H. Trinh, N. Cheng, J. Bradley, A. Chao, A. Mody, S. Truitt,
D. Metropolitansky, R. Ness, and J. Larson, "From Local to Global: A Graph
RAG Approach to Query-Focused Summarization," 2024.
https://arxiv.org/abs/2404.16130

[8] A. D. M. et al., "Are Complex Query Answering Benchmarks Really Complex?"
2025. https://openreview.net/forum?id=2FMdrDp3zI

[9] S. Yao, J. Zhao, D. Yu, N. Du, I. Shafran, K. R. Narasimhan, and Y. Cao,
"ReAct: Synergizing Reasoning and Acting in Language Models," *ICLR*, 2023.
https://arxiv.org/abs/2210.03629

[10] T. Schick, J. Dwivedi-Yu, R. Dessì, R. Raileanu, M. Lomeli, L. Zettlemoyer,
N. Cancedda, and T. Scialom, "Toolformer: Language Models Can Teach
Themselves to Use Tools," *NeurIPS*, 2023.
https://arxiv.org/abs/2302.04761

[11] J. Es, J. James, L. Espinosa-Anke, and S. Schockaert,
"RAGAS: Automated Evaluation of Retrieval Augmented Generation," *EACL*, 2024.
https://arxiv.org/abs/2309.15217

[12] J. Saad-Falcon, O. Khattab, C. Potts, and M. Zaharia, "ARES: An Automated
Evaluation Framework for RAG Systems," *NAACL*, 2024.
https://aclanthology.org/2024.naacl-long.20/
