# Enterprise Change Management RAG

Week 2 project for the Mastering Agentic AI course.

This project builds a Retrieval-Augmented Generation (RAG) assistant in n8n for enterprise Change Management questions. The knowledge base combines a project-created operational Change Management procedure with curated NIST and FedRAMP guidance.

## Architecture

- **Ingestion workflow:** file upload → document loader → 800-character chunks with 150-character overlap → OpenAI `text-embedding-3-small` → Pinecone
- **Retriever workflow:** user question → OpenAI embedding → Pinecone dense retrieval → top 12 chunks
- **Q&A workflow:** n8n chat → AI Agent → OpenAI `gpt-4o-mini` → retriever sub-workflow → grounded answer

## Repository structure

- `workflows/` — exported n8n workflows
- `corpus/` — project corpus used for ingestion
- `docs/` — evaluation and project documentation

## Final retrieval configuration

- Pinecone index: `enterprise-policies`
- Retrieval: dense
- Top-k: 12
- Reranking: off
- Embedding model: `text-embedding-3-small`
- Generation model: `gpt-4o-mini`
- Temperature: 0.1
- Indexed records: 37

## Evaluation

The bot was stress-tested with 15 questions covering direct lookups, NIST controls, FedRAMP/significant-change questions, multi-source synthesis, ambiguity, and unanswerable/live-system requests.

Working evaluation results:

- Retrieval quality: approximately **25/30**
- Strict faithfulness: **2/15 passes**

The key finding was that strong retrieval did not guarantee a faithful final answer. In several tests, relevant evidence was retrieved but not fully used during generation. Multi-source federal questions were the largest weakness.

See `docs/Week 2 Project — Enterprise Change Management RAG Evaluation & Documentation.pdf` for the full evaluation and failure analysis.

## Notes

The NIST and FedRAMP text files are curated ingestion extracts for this educational project and include canonical source URLs. The generic Change Management procedure is fictional project material and is not an official policy or compliance authority.
