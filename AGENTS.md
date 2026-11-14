# Project instructions

- Never mention or compare against the original/reference notebook inside `notebooks/ocr-finetune-vlm.ipynb`, including Markdown cells, comments, or outputs. Keep this notebook's explanations self-contained. The reference may still be consulted internally when the user requests adaptation.

- Before modifying the project, thoroughly analyze the relevant current code, applicable original-notebook sections, dependencies, execution order, and effects on existing behavior. Distinguish executable code from illustrative Markdown or comments, and implemented functionality from planned calls. Make changes only after identifying the concrete need; report verification and remaining uncertainties accurately.

- Update `README.md` whenever adding or changing a project feature. Explain its technical purpose, usage, dependencies, inputs, outputs, and relevant limitations.
- For every new or changed notebook code block, include useful code comments and an accompanying Markdown cell explaining its purpose, prerequisites, inputs, outputs, execution order, and relevant limitations. Explain API costs and image transmission when applicable.
- Write all code comments, explanatory Markdown, documentation, and generated status messages in English. Preserve Arabic source text, transcription, and task data in their original language.
- Keep the README aligned with the implemented code. Clearly distinguish working features from planned OCR, finetuning, and deployment capabilities.
- Follow the project priorities: local model inference, low budget, and document understanding beyond OCR transcription. Treat specific model choices, budgets, and extraction schemas as undecided until established with the user.
- The user selected `google/gemma-3-4b-it` after qualitative experiments. Preserve transcription fidelity: invented words are unacceptable for the legal documents. Document observed limitations and distinguish user-reported experiments from repository validation.

- When adapting user-supplied code, preserve its prompts and logic verbatim wherever compatible. Limit edits to required local paths, variable compatibility, English comments, and requested changes; do not rewrite prompts or add extra workflow checks without a concrete user request.
