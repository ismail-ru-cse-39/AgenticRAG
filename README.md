# Agentic RAG for Multi-Step Question Answering

**Research focus:** Retrieval Strategies, Architectural Evolution, Optimisation, Efficiency, and Cost–Quality Trade-offs.

## Overview

This repository contains research and working materials for a survey paper exploring the evolution of Retrieval-Augmented Generation (RAG) towards more adaptive and agentic architectures.

Traditional RAG systems typically retrieve relevant documents and use them as context for generating answers. However, complex questions often require multiple retrieval steps, query decomposition, evidence verification, and reasoning across different information sources.

Agentic RAG extends conventional RAG by introducing planning, iterative retrieval, reflection, tool selection, and adaptive decision-making. This research investigates how these capabilities improve multi-step and complex question answering while considering computational efficiency, system optimisation, and operational cost.

## Research Topics

The survey is organised around six main research themes:

1. **RAG for Multi-Step Queries**
   - Multi-hop question answering
   - Query decomposition and planning
   - Iterative retrieval and reasoning
   - Evidence aggregation and verification

2. **RAG for Complex Queries**
   - Query complexity detection
   - Adaptive retrieval strategies
   - Hierarchical and graph-based retrieval
   - Constraint-aware question answering

3. **Efficiency and Its Evolution in RAG Systems**
   - Retrieval efficiency
   - Reducing unnecessary LLM calls
   - Adaptive computation
   - Latency, token usage, and scalability

4. **Optimisation Techniques and Their Evolution**
   - Query rewriting and decomposition
   - Adaptive retrieval and reranking
   - Context compression
   - Retrieval correction and workflow optimisation

5. **Architectural Evolution of RAG Systems**
   - Traditional RAG architectures
   - Iterative and corrective RAG
   - Self-reflective and modular RAG
   - Graph-based and agentic RAG architectures

6. **Cost–Quality Trade-off in Agentic RAG**
   - Answer quality and faithfulness
   - Retrieval and generation evaluation
   - Computational and monetary costs
   - Quality-aware resource allocation

## Key Research Papers

The research considers influential approaches and evaluation frameworks, including:

- Retrieval-Augmented Generation (Lewis et al., 2020)
- IRCoT – Interleaving Retrieval with Chain-of-Thought Reasoning
- FLARE – Active Retrieval Augmented Generation
- Adaptive-RAG
- RAPTOR
- Self-RAG
- Corrective RAG (CRAG)
- Modular RAG
- LightRAG
- GraphRAG
- EfficientRAG
- MultiHop-RAG
- ARES
- RAGAS

## Research Objectives

- Understand how RAG systems have evolved from conventional retrieval pipelines to agentic architectures.
- Analyse how existing approaches address multi-step and complex queries.
- Compare retrieval, planning, reflection, and optimisation strategies.
- Investigate the relationship between answer quality, retrieval effectiveness, latency, and computational cost.
- Identify limitations and open research challenges.
- Explore opportunities for more efficient, adaptive, and evidence-aware Agentic RAG systems.

## Proposed Research Directions

The survey also explores potential improvements, including:

- Dependency-aware query decomposition for multi-step reasoning.
- Adaptive query-complexity estimation and retrieval routing.
- Evidence-sufficiency-based retrieval stopping.
- Cost-aware selection of models and retrieval tools.
- Modular, evidence-centred Agentic RAG architecture.
- Quality-constrained optimisation of retrieval and reasoning workflows.

These are proposed research directions to be investigated, rather than established results.

## Evaluation Dimensions

The research considers the following evaluation dimensions:

| Category | Example Metrics |
|---|---|
| Answer Quality | Exact Match, F1, Correctness |
| Retrieval Quality | Recall@K, Precision@K, MRR, NDCG@K |
| Grounding | Faithfulness, Citation Correctness |
| Reasoning | Sub-question Coverage, Evidence Sufficiency |
| Efficiency | Token Usage, LLM Calls, Retrieval Calls |
| Performance | Latency, Throughput |
| Cost | API Cost, Computational Cost |

## Repository Structure

```text
Agentic-RAG-Survey/
│
├── README.md
├── papers/
│   └── Research papers and references
│
├── literature-review/
│   └── Paper analysis and comparison notes
│
├── draft/
│   └── Agentic RAG survey paper drafts
│
└── notes/
    └── Research ideas and future directions
```

*The repository structure may evolve as the research progresses.*

## Research Status

This repository is an ongoing academic research project. It contains preliminary literature analysis, a working survey-paper draft, and proposed research directions. The literature and bibliographic details will be further verified and refined as the study progresses.

## Keywords

`Agentic RAG` `Retrieval-Augmented Generation` `Multi-Hop Question Answering` `Complex Queries` `Query Decomposition` `Adaptive Retrieval` `GraphRAG` `Self-RAG` `RAG Optimisation` `LLM Agents` `Retrieval Efficiency` `Cost-Quality Trade-off`

---

**Purpose:** Academic research and exploration of efficient, reliable, and adaptive Agentic RAG systems for multi-step and complex question answering.
