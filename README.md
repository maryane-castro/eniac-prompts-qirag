# QI-RAG Prompts

Reproducibility companion for **"Chunking Strategies for Retrieval in Enterprise Documents: A Comparative Study and the QI-RAG Proposal"**.

**Authors:** Maryane de Castro Lima¹, Thomaz Maia de Almeida²
¹ Instituto Federal de Educação, Ciência e Tecnologia do Ceará (IFCE)
² Núcleo de Visão Computacional e Engenharia (NUVEN)
Contact: maryane.castro@sti.ufc.br, thomaz.maia@ifce.edu.br

## About

The paper proposes **Question-Indexed RAG (QI-RAG)**, a retrieval strategy for
enterprise document RAG pipelines. Instead of indexing document chunks directly,
QI-RAG has an LLM generate several candidate questions per chunk and indexes those
questions instead — closing the semantic gap between colloquial user queries and
technical enterprise-document language. At query time, the user's question is
matched against this question index rather than against the raw chunks; if no match
clears a similarity threshold, the pipeline falls back to a direct search over the
original chunks. The approach runs in four phases (chunking and type detection,
question generation, question indexing, hybrid retrieval with fallback), described
in Sections 3.4–3.7 of the paper.

This repository publishes the actual prompt templates (Gemma 3) used to run that
pipeline — figure description, question generation (textual/tabular/figure chunks),
final answer synthesis, the GEval correctness judge, and the GEval retrieval-accuracy
judge — since the paper itself describes prompt *behavior* but not the literal prompt
text.

See [`prompt.md`](./prompt.md) for the full set of prompts and implementation notes.

## Generation Parameters

The same configuration was used across all LLM calls in the pipeline (Gemma 3
question/description/answer generation and the GPT-OSS 120B GEval judges), unless
noted otherwise in `prompt.md`:

| Parameter | Value |
|---|---|
| Temperature | 0.5 |
| top-k | default (not overridden) |
| top-p | default (not overridden) |
| max_tokens | unlimited (no cap set) |

## Citation

BibTeX entry to be added once the camera-ready proceedings citation (venue, year,
pages, DOI) is finalized.

## License

Released under the [MIT License](./LICENSE).
