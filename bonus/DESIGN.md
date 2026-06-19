# Design Document: Vietnam E-Commerce Customer Support Feedback Flywheel and Knowledge-Graph-Augmented QA Pipeline

## 1. Problem Statement and Constraints
We are designing a production-grade data pipeline for an automated AI Customer Support Agent for a major Vietnamese e-commerce platform. The system handles 50,000 conversation turns daily, helping customers with queries ranging from generic policy FAQs (returns, warranty) to personalized order tracking and logistics.

The load-bearing constraints are:
1. **Vietnamese Language Nuances**: Queries contain regional dialects (Northern vs. Southern vocabulary), typos, missing tone accents, and informal abbreviations (e.g., "ko", "dc", "ship").
2. **Strict Data Privacy**: Compliance with Vietnam's **Decree 13/2023/ND-CP on Personal Data Protection (PDPD)**. All Personally Identifiable Information (PII) such as customer names, phone numbers, and home addresses must be scrubbed before logging traces to external LLMs.
3. **Latency and Compute Budgets**: Target end-to-end latency must be $< 2.0$ seconds. To reduce high operating costs of proprietary LLMs (OpenAI/Gemini), the system leverages a fine-tuned local open-source model (e.g., Llama-3-Vietnamese-8B), which requires a feedback flywheel to constantly learn from production telemetry.

---

## 2. Proposed Architecture

```
                                 [Customer Interactions]
                                            |
                                            v
                                   [AI Support Agent]
                                            | (OTel Spans Exported)
                                            v
                                  [Raw Spans (JSONL)] (Bronze)
                                            |
                                            v
                                  [PII Redaction Gate] (Regex + NER)
                                            |
                                            v
                                [Cleaned Bronze Spans]
                                            |
                         +------------------+------------------+
                         |                                     |
                         v (Batch ETL: 1 hour)                 v (Incremental Graph Ingestion)
               [Silver: Deduplicated]                    [Knowledge Graph] (DuckDB/Neo4j)
                         |                                     |
        +----------------+----------------+                    | (Orders -> Category -> Warehouse)
        |                                 |                    v
        v (Success turns)                 v (Failed turns)   [Multi-hop Logistics Query]
[Candidate Chosen]                [Candidate Rejected]
        |                                 |
        +----------------+----------------+
                         | (Join by user prompt)
                         v
               [Preference Pairs (Raw)]
                         |
                         v (Semantic Decontamination: cosine > 0.85) <--- [Evaluation Set]
               [Decontaminated DPO Pairs]
                         |
                         v
              [Model SFT / DPO Training]
```

---

## 3. Core Architectural Decisions and Tradeoffs

### Question 1: Source, Shape, and PII Gate (Privacy vs. Redaction Overhead)
- **Decision**: Perform PII scrubbing at the ingest stage (Bronze to Silver transition) using a hybrid system: hardcoded Regex for Vietnamese phone formats, and a lightweight, local named-entity recognition (NER) model (e.g., fine-tuned PhoBERT-NER) for names/addresses.
- **Tradeoff (Regex+NER vs. LLM-based redaction)**: 
  - *Regex+NER (Chosen)*: Processes each conversation turn in under $10\text{ms}$ with zero API cost. While it can miss creative/slang name structures, it catches $98\%$ of standard format PII.
  - *LLM Redaction (Rejected)*: Provides near-perfect accuracy but adds $500\text{ms}$ latency and incurs massive API token costs.
- **Reasoning**: Latency is a hard constraint for user chat. Adding an extra LLM call just for PII sanitization is economically and temporally unviable.

### Question 2: Batch vs. Streaming for Flywheel (Freshness vs. Infrastructure Complexity)
- **Decision**: Implement a **1-hour micro-batch pipeline** using DuckDB to process traces and generate preference pairs.
- **Tradeoff (Real-time Streaming vs. Micro-batch)**:
  - *Real-time Streaming (Rejected)*: Ingesting traces via Kafka and matching chosen/rejected pairs instantly using stateful streaming (Flink) requires complex cluster maintenance and high standby costs.
  - *Micro-batch (Chosen)*: Simple, hourly cron job executing DuckDB queries. It provides a feedback loop fast enough to catch daily anomalies (e.g. if the agent hallucinates after a system update) while keeping infrastructure lightweight and running on standard VM resources.

### Question 3: Train/Serve Parity and Point-in-Time Features
- **Decision**: Use `ASOF JOIN` in DuckDB/dbt to construct features such as `historical_order_count` and `aggregate_refund_amount` at the exact time of the user query when curating DPO/SFT training records.
- **Tradeoff (Naive Join vs. ASOF Join)**:
  - *Naive Join (Rejected)*: Joining the current customer statistics (e.g., total spend in June) with a conversation turn from May leaks future purchases. The model learns to act differently for users it "knows" will spend more in the future, rendering it useless at serving time.
  - *ASOF Join (Chosen)*: More complex to write but guarantees train-serve parity by strictly showing the model the customer's state exactly as it was when they initiated the chat.

### Question 4: Flywheel Decontamination (Exact vs. Semantic Matching)
- **Decision**: Perform semantic decontamination by calculating the cosine similarity of sentence embeddings (using a local sentence-transformer) between the training prompts and the evaluation set, dropping any pair with similarity $> 0.85$.
- **Tradeoff (Exact Match vs. Semantic Match)**:
  - *Exact Match (Rejected)*: In Vietnamese e-commerce, users ask the same question in dozens of paraphrased ways (e.g., "giao hàng từ đâu" vs. "đơn hàng được gửi từ kho nào"). Exact matching would fail to detect this, leaking evaluation intent into the training set.
  - *Semantic Match (Chosen)*: Prevents semantic leakage of the evaluation set, ensuring evaluation metrics remain a true measure of generalization.

### Question 5: Unstructured Knowledge - Vector Database vs. Knowledge Graph
- **Decision**: A hybrid system: use flat Vector RAG for general return policies, and a Knowledge Graph (constructed in DuckDB/Neo4j) for logistics and inventory questions.
- **Tradeoff (Pure Vector RAG vs. Hybrid KG)**:
  - *Pure Vector RAG (Rejected)*: Struggles to join facts across separate chunks (e.g., "Sản phẩm A thuộc danh mục B" and "Danh mục B được vận chuyển từ Kho Hà Nội"). Top-K retrieval only returns one chunk, causing retrieval failure.
  - *Hybrid KG (Chosen)*: Walking the graph (`Product A` -> `Category B` -> `Hanoi Warehouse`) resolves multi-hop questions reliably with zero hallucination.

---

## 4. Rejected Alternative
- **Rejected Alternative**: Using a stateless LLM classifier to filter out toxic or incorrect answers in real-time, rather than building an offline feedback flywheel.
- **Reason for Rejection**: Evaluating the model's performance in real-time using another LLM increases the cost per turn exponentially and adds significant latency. A feedback flywheel turns telemetry into training data, allowing us to offline fine-tune the model to become inherently better and safer, eliminating the need for expensive online guardrail LLMs.

---

## 5. Vietnamese Context and Compliance
Under **Decree 13/2023/ND-CP**, e-commerce operators in Vietnam face severe penalties for transferring personal data out of Vietnam without explicit consent. By running the PII Redaction Gate on local virtual machines (within Vietnam cloud regions) and only exporting de-identified prompts, we maintain full regulatory compliance. Furthermore, using a local embedding model optimized for Vietnamese ensures high semantic accuracy during the decontamination phase.
