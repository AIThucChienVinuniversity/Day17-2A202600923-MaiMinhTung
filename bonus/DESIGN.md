# Bonus Design — Data Flywheel for a Vietnamese Customer Support Chatbot

## 1. Problem Statement

I want to design a data pipeline for a Vietnamese customer support chatbot used by an e-commerce platform. The chatbot answers questions about return policies, warranties, shipping status, payment issues, and product information. The system serves both customers and support operators.

The challenge is that production conversations are highly unstructured. Users may type Vietnamese with or without accents, mix Vietnamese and English in the same sentence, ask incomplete questions, or refer to previous conversation turns. The chatbot may also interact with external tools such as order lookup systems, warranty databases, and product catalogs.

The goal of this pipeline is not only to store logs but also to transform real production traffic into high-quality evaluation datasets and fine-tuning datasets. The pipeline should continuously improve the chatbot through a feedback loop while preventing data leakage and maintaining data quality.

---

## 2. High-Level Architecture

```text
Customer Conversations
          │
          ▼
   Raw Agent Traces
          │
          ▼
 Bronze Storage (JSON)
          │
          ▼
 Validation + PII Filtering
          │
          ▼
 Silver Clean Traces
          │
    ┌─────┴───────────┐
    ▼                 ▼
 Eval Dataset     Preference Pairs
    │                 │
    └─────┬───────────┘
          ▼
   Human Review Layer
          │
          ▼
 Gold Training Dataset
          │
          ▼
      SFT / DPO
          │
          ▼
     New Model
          │
          ▼
 Production Traffic
          │
          └──────────► Flywheel Continues
```

---

## 3. Design Question 1 — What Are the Data Sources and How Messy Are They?

The primary source is production chatbot traces. Each trace contains:

* User question
* Retrieved documents
* Tool calls
* Model response
* Token usage
* Latency
* User feedback

The schema is expected to evolve over time. New tools may be introduced, additional metadata may appear, and existing fields may change.

Because schema drift is unavoidable, I would store all raw traces in a Bronze layer as append-only JSON. This preserves the original source of truth and allows future reprocessing.

In the Silver layer, I would normalize important fields such as:

* trace_id
* user_input
* agent_output
* latency_ms
* token_count
* feedback_score
* outcome

### Trade-off

**Option A: Store everything in structured tables immediately**

Pros:

* Easy querying
* Simpler analytics

Cons:

* Fragile when schema changes

**Option B: Raw JSON in Bronze, structured Silver**

Pros:

* Flexible
* Supports schema evolution

Cons:

* More processing complexity

I choose Option B because production systems inevitably change.

---

## 4. Design Question 2 — Batch or Streaming?

Monitoring and dataset generation have different requirements.

For operational monitoring, near-real-time visibility is useful. If tool failures suddenly spike, engineers should know immediately.

For training dataset generation, real-time processing is unnecessary. Building datasets every night is sufficient.

### Trade-off

**Streaming Pipeline**

Pros:

* Fresh data
* Faster feedback

Cons:

* More operational complexity
* Higher infrastructure cost

**Nightly Batch Pipeline**

Pros:

* Simpler
* Easier debugging
* Lower cost

Cons:

* Less fresh

I would use a hybrid approach:

* Streaming for monitoring
* Batch for dataset generation

This provides freshness where necessary while keeping training workflows stable.

---

## 5. Design Question 3 — How Do We Prevent Data Leakage?

Data leakage is one of the most dangerous failure modes in AI systems.

If a prompt appears in both the evaluation dataset and the training dataset, evaluation scores become misleading.

To prevent this, I would implement a decontamination step before creating training data.

The first stage would remove exact prompt matches after normalization:

* Lowercasing
* Whitespace normalization
* Unicode normalization

The second stage would use fuzzy matching based on embeddings to detect rewritten versions of evaluation prompts.

### Example

Evaluation prompt:

> Can I return a widget after 10 days?

Training prompt:

> Is a widget still refundable if purchased 10 days ago?

Although the text differs, the semantic meaning is nearly identical. Embedding similarity can identify and remove such cases.

I consider leakage prevention mandatory because it directly impacts model evaluation reliability.

---

## 6. Design Question 4 — Vector RAG or Knowledge Graph?

Many customer support questions can be answered using vector retrieval.

Example:

> What is the warranty period for a gadget?

A simple semantic search usually works well.

However, some questions require combining multiple facts.

Example:

> Which returnable products are shipped from the Hanoi warehouse?

This requires:

```text
widget
   ↓
accessory
   ↓
Hanoi warehouse
```

No single document chunk may contain the entire reasoning chain.

### Trade-off

**Vector RAG**

Pros:

* Easy implementation
* Strong semantic retrieval

Cons:

* Weak for multi-hop reasoning

**Knowledge Graph**

Pros:

* Handles relationships naturally
* Supports multi-hop queries

Cons:

* More complex ingestion

I would combine both:

* Vector retrieval for document search
* Knowledge graph for structured relationships

This hybrid approach balances accuracy and operational cost.

---

## 7. Design Question 5 — How Can We Build a Safe Flywheel?

The chatbot continuously generates new traces and feedback signals.

Potential signals include:

* Positive feedback
* Negative feedback
* Human corrections
* Escalation to support staff
* Retry behavior

These signals can be transformed into:

* Evaluation datasets
* DPO preference pairs
* Failure clusters

However, automatically training on all production outputs is dangerous.

Some successful responses may actually be hallucinations that users failed to notice.

Therefore, I would require at least one of the following:

* Human review
* Retrieval-grounded validation
* LLM-as-a-judge verification

before data enters the training dataset.

### Rejected Approach

I reject the strategy of training directly on all production conversations.

Reasons:

1. Production logs contain hallucinations.
2. Sensitive information may be present.
3. Evaluation prompts may leak into training.
4. User feedback is often noisy.

Although this approach is simple, it creates long-term model quality risks.

---

## 8. Cost and Operational Considerations

The most expensive components are:

* Embedding generation
* LLM-based evaluation
* Human review

To reduce costs, I would:

* Cache embeddings
* Re-embed only changed content
* Run expensive judges only on sampled conversations
* Prioritize conversations with negative feedback

This focuses resources where quality improvements matter most.

---

## 9. Vietnam-Specific Considerations

Vietnamese presents unique challenges:

* Missing accents
* Slang and abbreviations
* Code-switching between Vietnamese and English
* Unicode normalization issues

The pipeline should preserve original text while also storing normalized text for retrieval and evaluation.

Additionally, customer support logs often contain:

* Phone numbers
* Addresses
* Order identifiers

Under Vietnam's personal data protection regulations, these fields should be filtered or anonymized before entering training datasets.

---

## 10. Conclusion

This design uses a data flywheel architecture to continuously improve a Vietnamese customer support chatbot. The core principles are:

1. Preserve raw data in Bronze.
2. Enforce quality checks in Silver.
3. Build curated evaluation and training datasets.
4. Prevent train/eval leakage through decontamination.
5. Use point-in-time correctness for features.
6. Combine Vector RAG and Knowledge Graph for retrieval.
7. Never automatically trust production outputs.

The objective is not only to process data but to create a reliable system that continuously learns from real-world interactions while maintaining evaluation integrity and operational safety.
