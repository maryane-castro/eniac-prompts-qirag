# QI-RAG Prompts

Reproducibility companion for **"Chunking Strategies for Retrieval in Enterprise Documents: A Comparative Study and the QI-RAG Proposal"**.

**Authors:** Maryane de Castro Lima¹, Thomaz Maia de Almeida²
¹ Instituto Federal de Educação, Ciência e Tecnologia do Ceará (IFCE)
² Núcleo de Visão Computacional e Engenharia (NUVEN)
Contact: maryane.castro@sti.ufc.br, thomaz.maia@ifce.edu.br

## About

The paper proposes **Question-Indexed RAG (QI-RAG)**, which indexes LLM-generated
questions about each document chunk instead of the chunks themselves, closing the
semantic gap between colloquial user queries and technical enterprise-document
language. This repository publishes the actual prompt templates (Gemma 3) used to
run the pipeline described in Sections 3.4–3.7 of the paper — figure description,
question generation (textual/tabular/figure chunks), final answer synthesis, and the
GEval correctness judge — since the paper itself describes prompt *behavior* but not
the literal prompt text.

See [`prompt.md`](./prompt.md) for the full set of prompts and implementation notes.

## Citation

BibTeX entry to be added once the camera-ready proceedings citation (venue, year,
pages, DOI) is finalized.

## License

Released under the [MIT License](./LICENSE).
