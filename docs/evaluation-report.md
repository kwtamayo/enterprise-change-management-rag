# Week 2 Project — Enterprise Change Management RAG

## 1. Project Overview

I built a Retrieval-Augmented Generation (RAG) assistant for enterprise Change Management. The bot answers questions using a controlled corpus made up of an operational Change Management procedure plus targeted NIST and FedRAMP guidance. The goal is to produce concise, grounded answers and to refuse when the knowledge base does not contain enough evidence.

**One-line definition:** My RAG app helps Change Management practitioners answer production change, risk, configuration-control, and federal authorization questions from enterprise procedure, NIST, and FedRAMP guidance in an n8n chat interface, with retrieval and faithfulness measured against a 15-question evaluation set.

## 2. Build Track and Architecture

Build track: n8n no-code/low-code.

The system uses three workflows:

- **Ingestion:** browser file upload → document loader → 800-character chunks with 150-character overlap → OpenAI `text-embedding-3-small` → Pinecone.
- **Retriever:** user question → same OpenAI embedding model → dense Pinecone retrieval → top-k chunks returned to the agent.
- **Q&A:** n8n Chat Trigger → AI Agent → OpenAI `gpt-4o-mini` → KB Search Tool → retriever sub-workflow → grounded answer.

Final stable retrieval configuration:

- Pinecone index: `enterprise-policies`
- Dense retrieval
- Top-k: 12
- Reranking: off
- Embedding model: `text-embedding-3-small`
- Generation model: `gpt-4o-mini`
- Sampling temperature: 0.1
- Pinecone records after full corpus ingestion: 37

## 3. Corpus

The final corpus contains seven logical source units:

- Generic Operational Change Management Procedure
- NIST SP 800-53 targeted Change Management controls, including CM-2, CM-3, CM-4, CM-5, CM-6, and SI-2
- NIST SP 800-128 security-focused configuration management guidance
- NIST SP 800-37 Risk Management Framework change/authorization context
- FedRAMP 2026 Configuration Management guidance
- FedRAMP Significant Change Notification core rules
- FedRAMP Significant Change Notification category pathways

The NIST and FedRAMP files are curated ingestion extracts focused on the specific controls and rules needed for the evaluation rather than full copies of the publications.

## 4. Agent Instructions

The agent was instructed to always search the Change Management knowledge base before answering, pass the user’s full original question to retrieval, avoid narrowing the search to a single document, use all relevant retrieved sources when an answer spans multiple documents, and say clearly when the knowledge base does not contain enough evidence.

For federal, security, authorization, emergency, or compliance questions, the prompt tells the agent to consider applicable NIST and FedRAMP evidence in addition to the operational procedure.

## 5. Iterations

### Iteration 1 — Generic procedure only

The first working bot used only the Generic Operational Change Management Procedure. The architecture worked, but the first stress-test questions exposed a corpus problem: questions about NIST CM-3, FedRAMP, authorization impact, and emergency SCN rules could not be answered completely because those sources were not yet indexed.

### Iteration 2 — Expanded authoritative corpus

Six NIST/FedRAMP source units were added to the same Pinecone index, increasing the record count from 21 to 37. This materially improved answers, especially for direct NIST questions. CM-3 moved from an incomplete generic answer to a grounded answer citing the correct NIST source.

### Iteration 3 — Retrieval depth

The Pinecone retrieval limit was tested at 4, 8, and 12. Increasing top-k exposed more relevant evidence, but did not reliably improve final answers. In one emergency-change test, the relevant FedRAMP chunk appeared within the top 12 results but the agent still omitted it from the final answer.

### Iteration 4 — Prompt and tool-routing changes

The agent prompt and KB Search Tool description were revised so the full user question would be passed to retrieval and the agent would consider NIST/FedRAMP sources for federal and emergency questions. This improved use of operational evidence, but did not fully solve multi-source synthesis.

### Iteration 5 — Reranking attempt

n8n’s Pinecone reranking path was tested. The available reranker required a Cohere API credential. Because that introduced an additional external dependency and account setup, reranking was not implemented for the final no-code build. The final stable configuration returned to dense top-12 retrieval with reranking off.

## 6. Evaluation Method

The bot was stress-tested with 15 questions covering direct procedure lookups, NIST control questions, federal/FedRAMP questions, ambiguous requests, multi-source synthesis, and unanswerable/live-system questions.

Retrieval quality was scored on a 0–2 scale:

- 0 = unusable retrieval
- 1 = partial evidence
- 2 = sufficient evidence

Faithfulness used a strict pass/fail rule: the answer had to include all essential acceptance criteria, remain grounded in retrieved evidence, and avoid inventing unsupported facts.

## 7. Evaluation Results

Overall working scores:

- Retrieval quality: approximately **25/30**
- Strict faithfulness: **2/15 passes**

### Question-level summary

| # | Test | Retrieval | Faithfulness | Main issue |
|---|---|---:|---|---|
| 1 | Production authentication evidence | 1/2 | FAIL | FedRAMP screening incomplete |
| 2 | Rollback plan | 1/2 | FAIL | Some backup/configuration/access details missing |
| 3 | NIST CM-3 | 2/2 | PASS | Correct NIST source and required elements |
| 4 | Production network component replacement | 1/2 | FAIL | Architecture/boundary and FedRAMP screening incomplete |
| 5 | Emergency vulnerability | 2/2 | FAIL | FedRAMP emergency path retrieved but omitted from final answer |
| 6 | Temporary baseline deviation | 2/2 provisional | FAIL | Testing/monitoring details omitted |
| 7 | “Do I need approval?” | 2/2 provisional | FAIL | Should have asked for clarification |
| 8 | High-risk approval evidence | 2/2 provisional | FAIL | Residual-risk/FedRAMP treatment incomplete |
| 9 | Federal authorization / FedRAMP review | 1/2 | FAIL | Category-path and escalation details incomplete |
| 10 | “Testing passed. Can I approve?” | 2/2 provisional | FAIL | Should have requested missing context |
| 11 | NIST CM-4 | 2/2 | PASS | Correct CM-4 evidence |
| 12 | Changing security settings from baseline | 2/2 | FAIL | Missing current-vs-proposed and restoration/testing detail |
| 13 | Critical vulnerability synthesis | 1/2 | FAIL | FedRAMP SCN core source absent from top 12 |
| 14 | “Which specific people at my company?” | 2/2 provisional | FAIL | Should explicitly state corpus cannot identify internal people |
| 15 | Live change request CHG-004271 | 2/2 provisional | FAIL | Correctly refused to invent status, but system-access boundary could be clearer |

## 8. Failure Analysis

The largest lesson was that a high retrieval score does not guarantee a faithful final answer. Several questions retrieved enough evidence, but the model did not use all of it. The emergency-change case demonstrated this clearly: relevant FedRAMP material was present in the retrieved context, yet the final answer used only the generic operational procedure.

A second weakness was multi-source federal synthesis. Dense retrieval handled direct NIST control questions well, but questions requiring the procedure, NIST, and FedRAMP together were less reliable. Increasing top-k from 4 to 8 to 12 improved evidence availability but did not consistently improve the generated answer.

A third weakness was ambiguity and refusal behavior. The bot answered “Do I need approval?” and “The testing passed. Can I approve the change?” too directly rather than requesting missing context. It correctly refused to invent a live CHG status, but the refusal could be more explicit about system-access boundaries.

## 9. What I Learned

The most useful lesson was learning to separate corpus, retrieval, and generation failures instead of treating every wrong answer as a prompt problem. The first evaluation exposed a missing-corpus problem. After adding NIST/FedRAMP, direct retrieval improved. Later failures showed that relevant evidence can be retrieved and still be ignored during generation.

Top-k is not a quality knob that can be increased indefinitely. Returning more chunks can improve recall, but it can also add noise. The emergency case showed that simply moving from 4 to 8 to 12 did not guarantee better synthesis.

Reranking is a logical next improvement because it can promote the strongest evidence from a wider candidate set, but in this no-code implementation it introduced an additional Cohere dependency. Hybrid dense + keyword retrieval is another logical extension, especially for exact terms such as control IDs and FedRAMP rule names.

## 10. Current Limitations and Next Improvements

The final Week 2 build is a working dense RAG prototype, not a production-ready federal Change Management assistant. The highest-value next improvements would be hybrid dense + keyword retrieval, reranking, stronger ambiguity/refusal behavior, and more explicit source-level citations. A production version would also need live ITSM integration, access control, freshness automation, logging, and evaluation monitoring.

