# Benchmarking LLM Agents for Science: A Survey of Evaluation Frameworks

**Draft — March 2026**

---

## Abstract

The rapid deployment of large language model (LLM)-based agents in scientific workflows has created an urgent need for rigorous evaluation frameworks. Over the past two years, a proliferation of benchmarks has emerged to assess whether AI agents can perform meaningful scientific work—from reproducing published results to generating novel hypotheses. This survey provides a structured review of 23 benchmarks and evaluation frameworks for agentic science, organized into six categories: data-driven scientific discovery, research reproduction and replication, end-to-end research agents, ML engineering and competition, domain-specific scientific benchmarks, and scientific reasoning with agentic elements. We describe each benchmark's design, task structure, metrics, and key findings, and provide a comparative analysis across dimensions including domain coverage, evaluation methodology, and performance ceilings. Several cross-cutting themes emerge: current agents achieve at best 25–42% success on discovery tasks, reproduction remains substantially harder than reasoning, and scaling model size yields diminishing returns on science-specific evaluations even when general benchmarks continue to improve. We identify critical gaps in the current evaluation landscape, including limited coverage of experimental sciences, insufficient longitudinal evaluation, and the absence of benchmarks for collaborative or interdisciplinary research. This survey is intended as a reference for researchers designing new benchmarks and for scientists evaluating the readiness of AI agents for integration into research practice.

---

## 1. Introduction

The prospect of AI systems that can conduct scientific research—formulating hypotheses, designing experiments, writing and executing code, analyzing data, and synthesizing findings—has moved from speculative to plausible in a remarkably short time. Foundation models now routinely pass graduate-level science exams, generate competent research code, and produce text that is difficult to distinguish from human-authored scientific prose. The natural question is whether these capabilities compose into something resembling scientific competence.

Answering this question requires benchmarks that go beyond isolated capabilities. A model that can solve differential equations, write Python, and summarize papers may still fail catastrophically when asked to reproduce a published result or test a novel hypothesis against real data. The evaluation of *agentic* science—where systems must plan, execute multi-step workflows, interact with tools, and make decisions under uncertainty—demands fundamentally different assessment frameworks than those used for static question answering.

The period from 2024 to early 2026 has seen an explosion of such frameworks. At least 23 distinct benchmarks now target some aspect of scientific agency, ranging from narrowly scoped code reproduction tasks to ambitious end-to-end research simulations. These vary enormously in design philosophy, domain coverage, evaluation methodology, and ambition. Some grade binary task completion; others employ expert rubrics or compare against human leaderboards. Some restrict agents to a single tool call; others provide full computational environments with internet access.

This survey aims to impose order on this rapidly growing landscape. We organize existing benchmarks into a six-part taxonomy, describe each in sufficient detail for practitioners to assess relevance, provide a comparative table, and identify themes, gaps, and directions for future work. Our scope is benchmarks with an explicit agentic component—those requiring multi-step reasoning, tool use, or autonomous decision-making in scientific contexts—as of March 2026. We draw on three recent surveys of LLM agent evaluation (Mohammadi et al., 2025; Li et al., 2025; Wang et al., 2025) while focusing specifically on the scientific domain.

---

## 2. Taxonomy

We organize the benchmarks into six categories based on the *type of scientific activity* being evaluated, ordered roughly from specific capabilities to general competence:

1. **Data-Driven Scientific Discovery.** Benchmarks that evaluate whether agents can extract insights, generate hypotheses, or identify patterns from scientific datasets. These target the core epistemic activity of science: learning something new from evidence.

2. **Research Reproduction & Replication.** Benchmarks that ask agents to reproduce computational results from published papers. These test whether agents can understand a paper's methodology well enough to re-implement it, a necessary (though not sufficient) condition for scientific competence.

3. **End-to-End Research Agents.** Benchmarks that simulate the full research cycle—ideation, experimental design, implementation, execution, and analysis—typically within machine learning. These are the most ambitious evaluations, testing whether agents can function as autonomous researchers.

4. **ML Engineering & Competition.** Benchmarks grounded in competitive ML tasks (e.g., Kaggle), evaluating practical data science and engineering skills against human performance baselines.

5. **Domain-Specific Scientific Benchmarks.** Evaluations targeting particular scientific disciplines (biology, geoscience, mathematical modeling, symbolic regression), often designed with domain expert input.

6. **Scientific Reasoning & Knowledge Benchmarks.** Primarily static assessments (question-answering, problem-solving) that, while not fully agentic, establish performance ceilings on the foundational reasoning required for scientific agency.

This taxonomy is not perfectly clean—several benchmarks span categories, and the boundary between "reproduction" and "end-to-end research" is sometimes a matter of degree. We assign each benchmark to its primary category while noting overlaps.

---

## 3. Benchmark Descriptions

### 3.1 Data-Driven Scientific Discovery

**ScienceAgentBench** (Chen et al., 2024; ICLR 2025). Developed by the OSU NLP Group, ScienceAgentBench evaluates agents on data-driven discovery tasks extracted from peer-reviewed publications. Tasks require agents to interact with real scientific datasets, use appropriate analysis tools, and produce verifiable outputs. The benchmark emphasizes ecological validity by grounding tasks in actual published workflows. Evaluation uses both automated metrics and output verification against known results. The best-performing agent configurations achieve 32–42% task success rates, revealing substantial room for improvement. A notable design choice is the use of self-contained task specifications derived from papers, enabling reproducible evaluation without requiring the full paper context. *Limitations:* Tasks are curated from computational sciences, limiting generalization to experimental disciplines. arXiv:2410.05080.

**DiscoveryBench** (Majumder et al., 2024). The first benchmark to explicitly formalize multi-step data-driven discovery as a composition of hypothesis search and verification. DiscoveryBench provides both real scientific datasets and synthetic datasets with known ground-truth discoveries, enabling controlled evaluation. The multi-step structure—where agents must identify candidate hypotheses, design verification strategies, and assess evidence—captures the iterative nature of genuine discovery more faithfully than single-step evaluations. The best system achieves only 25% success, with failure analysis revealing that agents struggle most with the verification step. *Limitations:* Synthetic datasets, while enabling ground-truth evaluation, may not capture the messiness of real scientific data. arXiv:2407.01725.

**ResearchBench** (Liu et al., 2025). Decomposes scientific discovery into three evaluable sub-tasks: inspiration retrieval (finding relevant prior work), hypothesis composition (formulating testable claims), and hypothesis ranking (prioritizing among candidates). This decomposition enables fine-grained diagnosis of where in the discovery pipeline agents fail. ResearchBench draws on a corpus of published papers and their citation networks to construct tasks with verifiable ground truth. *Limitations:* The decomposition, while analytically useful, may miss emergent difficulties that arise only when sub-tasks are composed. arXiv:2503.21248.

**SDE Framework** (Du et al., 2025). The Scientific Discovery Evaluation framework spans four scientific domains and is designed to reveal discrepancies between performance on general-purpose benchmarks and actual scientific discovery capability. A key finding is that models showing steady improvement on standard benchmarks (MMLU, HumanEval) exhibit diminishing returns on SDE tasks as scale increases, suggesting that scientific discovery requires capabilities not well captured by conventional evaluations. *Limitations:* Four domains, while broader than most, still represent a small slice of science. arXiv:2512.15567.

### 3.2 Research Reproduction & Replication

**PaperBench** (Starace et al., OpenAI, 2025). One of the most ambitious reproduction benchmarks, PaperBench asks agents to replicate AI research papers by writing complete codebases from scratch. Evaluation combines expert-designed rubrics (assessing code structure, methodology fidelity, and scientific correctness) with execution verification (does the code run and produce comparable results?). The rubric-based approach enables partial credit and fine-grained analysis, distinguishing PaperBench from binary pass/fail evaluations. Results demonstrate that even frontier models struggle with full-paper reproduction, particularly for papers requiring novel architectural implementations.

**CORE-Bench** (Siegel, Kapoor et al., Princeton, 2024). A large-scale computational reproducibility benchmark comprising 270 tasks drawn from 90 published papers across computer science, social science, and medicine. CORE-Bench's distinctive contribution is its cross-disciplinary scope: by including papers from fields with different computational practices, it reveals how reproducibility challenges vary by domain. Tasks are tiered by difficulty, from running provided code with minor fixes to substantial re-implementation. *Limitations:* Focuses on computational reproducibility rather than experimental replication. arXiv:2409.11363.

**SciReplicate-Bench** (2025). Targets code reproduction from NLP papers specifically, with 100 tasks drawn from 36 publications. The narrow domain focus enables deeper evaluation of field-specific challenges, such as handling custom tokenizers, non-standard training loops, and paper-specific evaluation protocols. The best LLM achieves 39% execution accuracy—a figure that underscores how far agents are from reliable reproduction even in a computationally mature field. arXiv:2504.00255.

**LMR-Bench** (EMNLP 2025). Takes a distinctive approach to reproduction evaluation: rather than asking agents to reproduce entire papers, LMR-Bench masks specific functions in language modeling research codebases and asks agents to re-implement them. This "fill-in-the-blank" methodology isolates scientific method understanding from software engineering overhead, testing whether agents comprehend *why* a particular implementation choice was made, not just *what* code to write. arXiv:2506.17335.

**ReplicatorBench** (Center for Open Science, 2026). The newest entry in this category, ReplicatorBench extends reproduction evaluation to the social and behavioral sciences, a domain where replication crises have been particularly acute. A striking finding: agents can competently design replication experiments (generating protocols, power analyses, and pre-registration documents) but struggle significantly with resource retrieval—locating the specific materials, stimuli, and instruments required to actually run studies. This points to a gap between procedural knowledge and practical research infrastructure navigation. arXiv:2602.11354.

### 3.3 End-to-End Research Agents

**ResearchGym** (ICLR 2026). The most comprehensive end-to-end evaluation as of this writing, ResearchGym simulates the full AI research cycle on papers from ICML, ICLR, and ACL. Agents are given a research question and an executable codebase, then must ideate approaches, implement experiments, run them, analyze results, and produce findings. The benchmark reveals critical reliability gaps: agents frequently propose sound ideas but fail during implementation or misinterpret experimental results. arXiv:2602.15112.

**MLAgentBench** (Huang et al., Stanford; ICML 2024). An early and influential benchmark for ML experimentation, MLAgentBench evaluates the iterative research cycle: agents must design experiments, execute them, analyze outputs, and decide on next steps. The benchmark provides realistic ML tasks with well-defined improvement metrics, enabling quantitative comparison of agent strategies. Its influence is visible in several subsequent benchmarks that adopt similar iterative evaluation structures. arXiv:2310.03302.

**InnovatorBench** (2025). Distinguishes itself by evaluating *novel* research contributions rather than reproduction. Agents are assessed on their ability to produce genuinely new findings on open research questions, evaluated by domain experts. Tests include frontier models (Claude Sonnet 4, GPT-5, GLM-4.5, Kimi-K2), providing a snapshot of the state of the art. Results confirm that novelty generation remains substantially harder than reproduction or incremental improvement. arXiv:2510.27598.

**FML-Bench** (2025). Focuses on the breadth of exploration in automatic ML research, evaluating whether agents can survey a problem space, identify promising directions, and pursue multiple approaches rather than fixating on a single strategy. This emphasis on exploration—rather than exploitation of a known approach—targets a critical aspect of research competence that most other benchmarks neglect. arXiv:2510.10472.

**MLRC-Bench** (NeurIPS 2025). Introduces dynamic, performance-based evaluation through ML research challenges, avoiding the static task sets that can become saturated or memorized. By continuously generating new challenges, MLRC-Bench aims to provide a more durable evaluation signal. Published via OpenReview.

### 3.4 ML Engineering & Competition

**MLE-bench** (OpenAI, 2024). Constructs an offline Kaggle competition environment where agents are graded against real historical human leaderboards. This design provides a naturally calibrated difficulty scale and enables direct human-AI comparison. MLE-bench evaluates the full pipeline of competitive data science: data exploration, feature engineering, model selection, hyperparameter tuning, and submission formatting. It has become a standard evaluation for measuring practical ML engineering capability. arXiv:2410.07095.

### 3.5 Domain-Specific Scientific Benchmarks

**LAB-Bench** (FutureHouse, 2024). The Language Agent Biology Benchmark evaluates agents on practical biology research tasks, including literature search, experimental protocol design, data interpretation, and biological reasoning. LAB-Bench is notable for its comparison against human expert biologists, providing a meaningful performance baseline. Results show agents approaching human performance on some literature-based tasks while falling substantially short on tasks requiring experimental intuition. arXiv:2407.10362.

**GeoBenchX** (2025). Targets geospatial science with multi-step tasks drawn from real GIS practitioner workflows. Evaluates tool-calling capabilities (interacting with GIS software, databases, and APIs) in addition to reasoning, reflecting the tool-heavy nature of geospatial research. *Limitations:* Narrow domain, though representative of computational earth sciences more broadly. arXiv:2503.18129.

**MM-Bench / MM-Agent** (NeurIPS 2025). Comprises 111 problems from Mathematical Contest in Modeling (MCM) and Interdisciplinary Contest in Modeling (ICM) competitions, spanning physics, biology, and economics. These open-ended modeling problems require agents to formulate mathematical representations of real-world phenomena, a core scientific skill. A headline result: agent performance approaches that of human competition teams, suggesting that mathematical modeling may be among the first scientific capabilities where AI achieves parity. arXiv:2505.14148.

**LLM-SRBench** (ICML 2025, Oral). Evaluates scientific equation discovery—the ability to identify symbolic mathematical relationships in data. The best system achieves 31.5% symbolic accuracy. LLM-SRBench incorporates anti-memorization design, using novel equation forms unlikely to appear in training data, addressing a critical concern about benchmark contamination in symbolic reasoning evaluation. arXiv:2504.10415.

**AAAR-1.0** (Lou et al., 2025). Assesses three distinct research assistance capabilities: equation inference from context, experiment design given constraints, and identification of weaknesses in research papers. All tasks are expert-labeled, providing high-quality evaluation ground truth. The three-task structure enables profiling of agent strengths and weaknesses across different modes of scientific engagement.

### 3.6 Scientific Reasoning & Knowledge Benchmarks

**GPQA Diamond.** A set of graduate-level science questions requiring deep expert reasoning. While not agentic in the tool-use sense, GPQA Diamond establishes a ceiling on the factual and inferential knowledge available to agents, making it a useful reference point for interpreting performance on more complex agentic tasks.

**FrontierMath** (Epoch AI). An extremely challenging collection of mathematical problems, including problems at the boundary of current mathematical knowledge. FrontierMath tests the limits of formal reasoning and serves as an aspirational benchmark for mathematical research agents.

**COS LLM Benchmarking Project** (Center for Open Science). An ongoing, large-scale project funded by Open Philanthropy that evaluates agents across the full research lifecycle: replication, peer review, and research design. By spanning multiple research activities, the COS project aims to provide a holistic assessment of agent readiness for scientific integration. GitHub: CenterForOpenScience/llm-benchmarking.

---

## 4. Comparison Table

| Benchmark | Year | Venue | Domain | Tasks | Metric Type | Best Score | Open Source |
|-----------|------|-------|--------|-------|-------------|------------|-------------|
| ScienceAgentBench | 2024 | ICLR 2025 | Multi-science | 44 | Task success rate | 32–42% | Yes |
| DiscoveryBench | 2024 | — | Multi-science | Real + synthetic | Multi-step success | 25% | Yes |
| ResearchBench | 2025 | — | Multi-science | 3 sub-tasks | Sub-task accuracy | — | Yes |
| SDE Framework | 2025 | — | 4 domains | Multi-domain | Discovery metrics | — | Yes |
| PaperBench | 2025 | — | AI/ML | ~20 papers | Rubric + execution | — | Partial |
| CORE-Bench | 2024 | — | CS/SocSci/Med | 270 | Reproducibility rate | — | Yes |
| SciReplicate-Bench | 2025 | — | NLP | 100 | Execution accuracy | 39% | Yes |
| LMR-Bench | 2025 | EMNLP 2025 | Language modeling | Masked functions | Function accuracy | — | Yes |
| ReplicatorBench | 2026 | — | Social/Behavioral | Multi-study | Replication fidelity | — | Yes |
| ResearchGym | 2026 | ICLR 2026 | AI/ML | Multi-paper | End-to-end success | — | Yes |
| MLAgentBench | 2024 | ICML 2024 | ML | 13 | Improvement metrics | — | Yes |
| InnovatorBench | 2025 | — | LLM research | Novel tasks | Expert evaluation | — | Yes |
| FML-Bench | 2025 | — | ML | — | Exploration breadth | — | Yes |
| MLRC-Bench | 2025 | NeurIPS 2025 | ML | Dynamic | Performance-based | — | Yes |
| MLE-bench | 2024 | — | Data science | 75 | Kaggle percentile | — | Yes |
| LAB-Bench | 2024 | — | Biology | Multi-task | vs. human experts | — | Yes |
| GeoBenchX | 2025 | — | Geospatial | Multi-step | Task completion | — | Yes |
| MM-Bench | 2025 | NeurIPS 2025 | Math modeling | 111 | Contest scoring | ~Human | Yes |
| LLM-SRBench | 2025 | ICML 2025 | Symbolic regression | — | Symbolic accuracy | 31.5% | Yes |
| AAAR-1.0 | 2025 | — | Multi-science | 3 task types | Expert-labeled acc. | — | Yes |
| GPQA Diamond | 2024 | — | Graduate science | ~200 | QA accuracy | ~65% | Yes |
| FrontierMath | 2025 | — | Mathematics | — | Problem-solving | <5% | Partial |
| COS Project | 2025–26 | — | Multi-discipline | Ongoing | Lifecycle coverage | — | Yes |

---

## 5. Cross-Cutting Themes & Gaps

### 5.1 Common Findings

**Science is harder than benchmarks suggest.** A recurring theme is the disconnect between performance on general-purpose benchmarks and science-specific evaluations. The SDE Framework (Du et al., 2025) provides the most direct evidence: models that show continued improvement on MMLU and HumanEval exhibit diminishing returns on scientific discovery tasks. This suggests that the capabilities required for genuine scientific work—handling ambiguity, designing multi-step investigations, interpreting unexpected results—are not well captured by standard evaluations.

**Reproduction is harder than reasoning.** Agents that can answer graduate-level science questions (GPQA) or solve competition problems struggle dramatically when asked to reproduce published computational results. The gap between reasoning-in-isolation and reasoning-in-context (where "context" includes messy codebases, underdocumented data pipelines, and implicit methodological choices) is a central finding across the reproduction benchmarks.

**Verification is the bottleneck.** DiscoveryBench, ResearchGym, and several other benchmarks report that agents fail disproportionately at verification—checking whether a hypothesis is supported, whether code produces correct results, or whether experimental outcomes match expectations. This mirrors known challenges in AI alignment and suggests that scientific self-correction is a capability that warrants targeted development.

**Performance ceilings are low.** The best reported scores—25% on DiscoveryBench, 32–42% on ScienceAgentBench, 39% on SciReplicate-Bench, 31.5% on LLM-SRBench—indicate that current agents are far from reliable scientific tools. Even on reproduction tasks, which are arguably the easiest form of scientific work (since the answer is known), success rates rarely exceed 50%.

### 5.2 Gaps in the Evaluation Landscape

**Experimental sciences are underrepresented.** The overwhelming majority of benchmarks target computational work—running code, analyzing datasets, writing implementations. Benchmarks for wet-lab biology, chemistry synthesis, field ecology, or clinical research design are largely absent. LAB-Bench touches on biology but remains primarily literature-based. ReplicatorBench begins to address social science experimentation but focuses on study design rather than execution.

**No benchmarks for collaborative research.** Real science is collaborative. Researchers discuss ideas, divide labor, review each other's work, and negotiate interpretations. No current benchmark evaluates multi-agent scientific collaboration or human-AI teaming in research contexts.

**Longitudinal evaluation is missing.** Scientific research unfolds over weeks, months, and years. Current benchmarks evaluate tasks completable in minutes to hours. Whether agents can maintain coherent research programs over extended periods—revisiting failed approaches, building on partial results, adapting to new information—remains untested.

**Interdisciplinary work is absent.** Many important scientific problems require integration across domains. No benchmark evaluates whether agents can bridge disciplinary boundaries, translate concepts between fields, or combine methods from different traditions.

**Benchmark contamination is poorly addressed.** With the exception of LLM-SRBench's explicit anti-memorization design and MLRC-Bench's dynamic task generation, most benchmarks use static task sets drawn from published papers—exactly the kind of content that appears in training data. The extent to which reported performance reflects genuine capability versus memorization remains unclear.

**The ML/CS bias is extreme.** Of the 23 benchmarks surveyed, approximately 15 are primarily or exclusively focused on machine learning and computer science. Physics, chemistry, biology, earth sciences, and social sciences are marginally represented. This creates a distorted picture of agent readiness for science broadly.

---

## 6. Conclusion

The evaluation landscape for scientific AI agents has matured rapidly since 2024. We now have benchmarks spanning discovery, reproduction, end-to-end research, domain-specific skills, and foundational reasoning. The collective findings paint a sobering but instructive picture: current LLM agents possess impressive component capabilities but fall well short of reliable scientific competence. Success rates in the 25–42% range on discovery and reproduction tasks, coupled with evidence of diminishing returns from scaling, suggest that architectural or methodological innovations—not simply larger models—will be needed to close the gap.

The most critical directions for future benchmark development are: (1) expanding beyond ML/CS to cover experimental and interdisciplinary sciences, (2) developing evaluations for collaborative and longitudinal research, (3) addressing benchmark contamination through dynamic or procedurally generated tasks, and (4) creating benchmarks that evaluate not just task completion but the *quality of scientific reasoning*—whether agents pursue the right questions, design appropriate controls, and interpret results with appropriate uncertainty.

For practitioners considering the integration of AI agents into scientific workflows, the current benchmarks provide a useful calibration: agents are ready to assist with well-defined computational subtasks but not to be trusted with autonomous scientific judgment. The benchmarks reviewed here offer concrete tools for evaluating this boundary as models continue to improve.

---

## References

Chen, Z., et al. (2024). ScienceAgentBench: Toward Rigorous Assessment of Language Agents for Data-Driven Scientific Discovery. *ICLR 2025*. arXiv:2410.05080.

Du, Y., et al. (2025). SDE: A Framework for Scientific Discovery Evaluation. arXiv:2512.15567.

Huang, Q., et al. (2024). MLAgentBench: Evaluating Language Agents on Machine Learning Experimentation. *ICML 2024*. arXiv:2310.03302.

Li, J., et al. (2025). From Automation to Autonomy: A Survey on LLMs in Scientific Discovery. *EMNLP 2025*. HKUST-KnowComp.

Liu, H., et al. (2025). ResearchBench: Benchmarking LLMs in Scientific Discovery via Inspiration-Based Task Decomposition. arXiv:2503.21248.

Lou, R., et al. (2025). AAAR-1.0: Assessing AI's Potential to Assist Research. arXiv.

Majumder, B. P., et al. (2024). DiscoveryBench: Towards Data-Driven Discovery with Large Language Models. arXiv:2407.01725.

Mohammadi, F., et al. (2025). Evaluation and Benchmarking of LLM Agents: A Survey. *KDD 2025*. arXiv:2507.21504.

OpenAI. (2024). MLE-bench: Evaluating Machine Learning Agents on Machine Learning Engineering. arXiv:2410.07095.

Rein, D., et al. (2024). GPQA: A Graduate-Level Google-Proof Q&A Benchmark. arXiv.

Siegel, N., Kapoor, S., et al. (2024). CORE-Bench: Fostering the Credibility of Published Research Through a Computational Reproducibility Agent Benchmark. Princeton. arXiv:2409.11363.

Starace, J., et al. (2025). PaperBench: Evaluating AI's Ability to Replicate AI Research. OpenAI.

Wang, L., et al. (2025). Survey on Evaluation of LLM-based Agents. arXiv:2503.16416.

Center for Open Science. (2026). ReplicatorBench: Evaluating LLM Agents for Replicability in Social and Behavioral Sciences. arXiv:2602.11354.

ResearchGym Authors. (2026). ResearchGym: Benchmarking End-to-End AI Research Agents. *ICLR 2026*. arXiv:2602.15112.

FutureHouse. (2024). LAB-Bench: Measuring Capabilities of Language Models for Biology Research. arXiv:2407.10362.

GeoBenchX Authors. (2025). GeoBenchX: Benchmarking LLM Agents on Multi-Step Geospatial Tasks. arXiv:2503.18129.

MM-Bench Authors. (2025). MM-Agent: Benchmarking LLM Agents on Mathematical Modeling Contest Problems. *NeurIPS 2025*. arXiv:2505.14148.

LLM-SRBench Authors. (2025). LLM-SRBench: A New Benchmark for Scientific Equation Discovery with LLMs. *ICML 2025 (Oral)*. arXiv:2504.10415.

InnovatorBench Authors. (2025). InnovatorBench: Evaluating LLM Agents on End-to-End Novel Research. arXiv:2510.27598.

FML-Bench Authors. (2025). FML-Bench: Benchmarking Automatic Machine Learning Research Agents. arXiv:2510.10472.

SciReplicate-Bench Authors. (2025). SciReplicate-Bench: Evaluating LLMs on Code Reproduction from NLP Papers. arXiv:2504.00255.

LMR-Bench Authors. (2025). LMR-Bench: Benchmarking LLMs on Language Modeling Research Reproduction. *EMNLP 2025*. arXiv:2506.17335.

MLRC-Bench Authors. (2025). MLRC-Bench: Dynamic Benchmarking for ML Research Challenges. *NeurIPS 2025*. OpenReview.

Epoch AI. (2025). FrontierMath: A Benchmark for Evaluating Advanced Mathematical Reasoning.

Center for Open Science. (2025–2026). COS LLM Benchmarking Project. GitHub: CenterForOpenScience/llm-benchmarking.
