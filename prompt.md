# Prompts for Reproducing the QI-RAG Pipeline

This document collects the prompts needed to faithfully reproduce each step described in the paper *"Chunking Strategies for Retrieval in Enterprise Documents: A Comparative Study and the QI-RAG Proposal"*.

---

## Phase 0 — Preprocessing (Docling)

Not an LLM prompt: Docling converts each PNG page of the dataset into structured Markdown, preserving tables in `|col|col|` syntax and marking non-transcribable figures with `<!-- image -->`. This step is deterministic/algorithmic, not generative.

---

## Phase 1 — Chunking and Type Detection

Chunking via `RecursiveCharacterTextSplitter` (800 characters, 200 overlap) — also deterministic. Type classification (textual / tabular / figure) is done via regex over the Markdown:
- Tabular: presence of a `|---|` separator row
- Figure: presence of the `<!-- image -->` marker
- Textual: otherwise (default)

### Prompt — Figure Description (chunk type "figure")

Used when Docling inserts `<!-- image -->`. Gemma 3 generates a textual description from the surrounding context (section title, preceding paragraph), which becomes an additional chunk `c_desc`.

```
You are an assistant specialized in corporate financial documents.

A figure (chart, image, or other non-transcribable visual element) was found in
the following excerpt of a financial report. Based ONLY on the textual context
provided below — the section title and the paragraph(s) immediately preceding
the figure — generate an objective description of what this figure likely
represents.

Section title: {section_title}
Preceding paragraph(s): {preceding_paragraph}

Instructions:
- Describe the likely type of figure (bar chart, line chart, pie chart, diagram,
  etc.) if there is sufficient textual evidence.
- Describe which quantities, periods, or categories are likely represented,
  based on the context.
- Do NOT invent specific numeric values that are not present in the context.
- If the context is insufficient to infer the content, describe only what can
  reasonably be stated (e.g., "chart related to [section topic]").
- Write a single paragraph, 2 to 4 sentences, in neutral technical language.

Figure description:
```

### Prompt — Table Header Preservation (not an LLM step)

Deterministic rule: when a table is split across multiple segments, the original header row (`|col|col|` + separator) is replicated at the top of each derived segment via string manipulation, not generation.

---

## Phase 2 — Question Generation (core of QI-RAG)

For each chunk `c_i` (textual, tabular, or figure description), Gemma 3 generates `k_i ≥ 5` semantically diverse questions, covering 4 types: definition, numerical value, comparison, process. The prompt varies by chunk type.

### 2a. Prompt — Textual Chunk

```
You are an expert at creating search questions for a retrieval-augmented
generation (RAG) system over corporate financial reports.

Given the document excerpt below, generate questions in NATURAL and
COLLOQUIAL language — as a real user would ask, not as the document is
written — that could be answered using ONLY the information contained in
this excerpt.

Excerpt:
"""
{chunk_text}
"""

Requirements:
- Generate AT LEAST 5 questions.
- If the excerpt contains more than 5 distinct, relevant facts, generate
  additional questions (one per additional fact) — do not truncate or force a
  fixed number.
- Distribute the questions across these 4 types, covering at least one of
  each when the excerpt's content allows:
  1. Definition — "What is X?" / "What does X mean?"
  2. Numerical value — "What was the value of X in period Y?"
  3. Comparison — "How did X change between period Y and Z?"
  4. Process — "How is X calculated/determined?"
- Each question must be self-contained (do not use "this", "the excerpt
  above", etc.).
- Use colloquial business language, avoiding the document's technical jargon
  when a more natural question is possible (e.g., "how much did the company
  spend" instead of "what was the consolidated expenditure").
- Do not generate questions whose answer is not present in the excerpt.

Respond ONLY in JSON format, with no additional text, in the following shape:
{
  "questions": [
    {"type": "definition", "question": "..."},
    {"type": "numerical_value", "question": "..."},
    {"type": "comparison", "question": "..."},
    {"type": "process", "question": "..."}
  ]
}
```

### 2b. Prompt — Tabular Chunk (prioritizes quantitative/comparative questions)

```
You are an expert at creating search questions for a retrieval-augmented
generation (RAG) system over corporate financial reports.

The excerpt below is a TABLE (or table fragment) in Markdown format, extracted
from a financial report. The original header has been preserved to keep each
cell interpretable.

Table:
"""
{chunk_text}
"""

Requirements:
- Generate AT LEAST 5 questions, prioritizing the QUANTITATIVE and
  COMPARATIVE types, since the content is tabular:
  1. Numerical value — ask about specific row/column values
     (e.g., "What was [metric] in [period]?")
  2. Comparison — ask about variation across columns/periods/categories
     (e.g., "How did [metric] change between [period A] and [period B]?")
  3. Definition — only if some term in the table requires explanation
  4. Process — only if applicable (e.g., how a derived metric is calculated
     from other rows in the table)
- Generate additional questions beyond 5 if the table covers more than 5
  relevant row/column/period combinations — do not truncate.
- Always reference the metric/row name and the period/column explicitly in
  the question; never use vague references like "that value".
- Use colloquial language, as an analyst would ask in a meeting, not as the
  table's technical header is phrased.

Respond ONLY in JSON format, with no additional text, in the following shape:
{
  "questions": [
    {"type": "numerical_value", "question": "..."},
    {"type": "comparison", "question": "..."}
  ]
}
```

### 2c. Prompt — Figure Description Chunk

```
You are an expert at creating search questions for a retrieval-augmented
generation (RAG) system over corporate financial reports.

The excerpt below is a GENERATED DESCRIPTION of a figure/chart (the original
figure could not be transcribed directly; this is a textual description of
what it represents, based on the surrounding context).

Figure description:
"""
{chunk_text}
"""

Requirements:
- Generate AT LEAST 5 questions, prioritizing:
  1. Trends — "What was the trend of X over time?"
  2. Highlighted values — "What was the maximum/minimum value of X shown in
     the chart?"
  3. Visual relationships — "How does X compare to Y in the chart?"
  4. Definition/context — "What does this chart represent?"
- Questions should reflect what a user would ask when looking at the chart,
  not the textual description itself.
- Do not invent precise numeric data that is not present in the description.

Respond ONLY in JSON format, with no additional text, in the following shape:
{
  "questions": [
    {"type": "trend", "question": "..."},
    {"type": "highlighted_value", "question": "..."}
  ]
}
```

---

## Phase 3 — Question Indexing

Not an LLM prompt: each generated question is vectorized with `ϕ` (BGE-M3 or Nomic Embed v1.5) and stored in ChromaDB with a pointer (`chunk_id`) to the source chunk. Purely algorithmic step.

---

## Phase 4 — Hybrid Retrieval with Fallback

Also algorithmic (cosine similarity search, threshold `τ = 0.75`, `n = 4`), with no text generation until the final step:

1. `ϕ(user_query)` is compared against the collection of indexed questions.
2. If `top1_similarity ≥ 0.75` → retrieve the 4 most similar questions → return their linked source chunks (primary path).
3. Otherwise → search directly over the original chunk collection, return the 4 closest chunks (fallback).

### Prompt — Final Answer Generation (Gemma 3, all strategies)

Used for both the baselines and QI-RAG, at the response-generation step (Level 2 evaluation).

```
You are an assistant specialized in analyzing corporate financial reports.
Answer the user's question using ONLY the information contained in the
retrieved context below. Do not use external knowledge or make assumptions
beyond what is explicitly present in the context.

Retrieved context:
"""
{retrieved_chunks_context}
"""

User question: {query}

Instructions:
- If the context contains the necessary information, answer directly,
  objectively, and completely, citing the exact values/terms from the context
  when applicable.
- If the necessary information is NOT in the context, explicitly state that
  the provided document does not contain that information — do not guess.
- Do not include comments about the search process or the retrieved chunks;
  answer only the content relevant to the question.
- Keep the answer concise (a few sentences), unless the question requires a
  process/calculation explanation, in which case it may be more detailed.

Answer:
```

---

## Phase 5 — Evaluation (Level 1 and Level 2, DeepEval / RAGAS / LLM-as-judge)

The paper uses DeepEval with RAGAS metrics (Level 1, computed automatically over embeddings/overlap, requiring no custom prompt beyond DeepEval's internal templates) and GEval (Level 2, correctness) with GPT-OSS 120B via Groq as judge.

### Prompt — GEval Correctness (judge: GPT-OSS 120B)

Reconstruction of the GEval correctness criterion with chain-of-thought, following the standard DeepEval pattern:

```
You are an expert, impartial evaluator of answers generated by
question-answering systems over corporate financial documents.

Your task is to evaluate the CORRECTNESS of the generated answer, comparing
it against the reference answer (ground truth), on a scale from 0.0 to 1.0.

Question: {query}

Reference answer (ground truth): {reference_answer}

System-generated answer: {generated_answer}

Evaluation criteria (reason step by step before giving the final score):
1. Does the generated answer contain the essential factual information present
   in the reference answer (numeric values, names, periods, conclusions)?
2. Does the generated answer contradict any fact present in the reference
   answer?
3. Does the generated answer omit critical information present in the
   reference?
4. Additional information in the generated answer that is not in the
   reference is acceptable as long as it does not contradict the reference
   and is plausibly correct given the financial domain context.

First, write your step-by-step reasoning comparing the two answers. Then, on
the last line, give the final score ONLY in the format:
SCORE: <number between 0.0 and 1.0>

Reasoning:
```

### Notes on the remaining generation metrics

- **Answer Relevance**: typically computed via DeepEval's default template (decomposes the answer into statements and checks pertinence to the question via embeddings/LLM) — requires no custom prompt beyond the library's native template.
- **BERTScore, METEOR, ParaScore**: purely algorithmic metrics (no LLM involved), computed over BERT embeddings / n-grams / sentence-transformer similarity between the generated and reference answers.
- **Contextual Precision/Recall/Relevance (RAGAS)**: use DeepEval/RAGAS's internal templates, which have the LLM judge, chunk by chunk, whether each retrieved excerpt is relevant to the question and whether the reference information is covered — reconstruct from the official DeepEval documentation (`ContextualPrecisionMetric`, `ContextualRecallMetric`, `ContextualRelevancyMetric`) if you want the exact text used by these classes, since the paper does not republish them.

---

## Implementation Notes

- Prompts 2a–2c request **strict JSON** output — fallback parsing is recommended (try `json.loads`, and if it fails, resend asking for a format correction), since the paper reports variable question counts per chunk (5–9 for tables/figures, potentially more for long text chunks), which requires generation without an overly restrictive `max_tokens`.
- The `k_min = 5` parameter should be enforced via *retry* if the model returns fewer than 5 questions.
- For the 3 Gemma 3 size variants (4B/12B/27B), the same question-generation and final-answer prompts can be reused — the paper does not report different prompts per model size, only the same pipeline applied to each checkpoint.

### Structured output by model size (Gemma 3 4B vs. 12B/27B)

Important difference in the actual implementation: for the **12B and 27B** models, **structured output** was used (JSON guaranteed/validated by the API call itself, via native `response_format`/JSON schema), whereas for **Gemma 3 4B** this structured output could not be used, so prompts 2a–2c are used **exactly as written** (with the explicit JSON-format instruction at the end of the prompt), and extraction relies on fallback parsing:

```python
import json, re

def parse_questions_output(raw_text: str) -> dict:
    # try parsing directly
    try:
        return json.loads(raw_text)
    except json.JSONDecodeError:
        pass
    # try extracting the first {...} block from the text (in case the model
    # added extra text before/after the JSON, common without structured output)
    match = re.search(r"\{.*\}", raw_text, re.DOTALL)
    if match:
        try:
            return json.loads(match.group(0))
        except json.JSONDecodeError:
            pass
    return None  # triggers a retry: resend asking to "respond only with valid JSON"
```

If `parse_questions_output` returns `None` or the question list has fewer than 5 items, resend the call with a correction prompt, for example:

```
Your previous response was not valid JSON (or had fewer than 5 questions).
Respond AGAIN, with only the valid JSON in the specified format, with no text
before or after, and with at least 5 questions.
```
