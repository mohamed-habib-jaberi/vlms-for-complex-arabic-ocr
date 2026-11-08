# VLMs for Complex Arabic OCR

## Master VLM Finetuning: Extracting Data from Challenging Arabic Legal Documents

We dive deep into the world of Vision Language Models (VLMs) to solve a real-world problem: processing low-quality, scanned Arabic legal documents. Standard OCR often fails with complex layouts and poor scan quality; here, we leverage the reasoning power of VLMs to extract structured data with high precision.

## Project context and priorities

1. **Local model:** target a VLM that can run on local hardware, with model weights available locally rather than requiring a hosted inference API. Model selection must account for available memory, supported hardware, and inference speed. Local inference is a goal; it is not implemented yet.
2. **Low budget:** prioritize small models, efficient image preparation, and parameter-efficient finetuning such as LoRA. Evaluate a baseline before spending on training, and track compute costs alongside quality. A specific spending limit and hardware budget have not been set.
3. **More than an OCR model:** aim to understand document content and layout and extract structured legal information, beyond transcribing text. Candidate tasks include identifying document types, extracting fields such as decision numbers and dates, and answering questions grounded in document pages. These are proposed capabilities; their output schema and evaluation criteria will be defined before implementation.

The current PDF-to-image pipeline provides inputs for this future VLM workflow. Model choice, training configuration, and deployment must follow these priorities.

## What you will learn

- **Data Strategy:** How to parse complex PDFs and generate synthetic data for niche tasks.
- **The Stack:** Using LlamaFactory for finetuning, Vast.ai for GPU compute, and WandB for experiment tracking.
- **Deployment:** Merging LoRA weights and optimizing inference using vLLM.

## Candidate vision-language models

The initial shortlist includes these three model families. They are candidates for evaluation, not a verified ranking for Arabic legal OCR.

| Model family | Reference | Selection considerations |
| --- | --- | --- |
| Qwen3-VL | [Hugging Face collection](https://huggingface.co/collections/Qwen/qwen3-vl) | Compare a small vision-language checkpoint on Arabic transcription, document understanding, and structured extraction. Check memory needs and compatibility with the chosen runtime. |
| LLaVA-NeXT | [Hugging Face collection](https://huggingface.co/collections/llava-hf/llava-next) | Select an exact checkpoint and evaluate its Arabic capabilities; these depend on the underlying language model and training data. |
| Gemma 3 | [Hugging Face collection](https://huggingface.co/collections/google/gemma-3-release) | Select a vision-capable variant: the Gemma 3 1B model is text-only, while the 4B, 12B, and 27B variants support image inputs. Check checkpoint access requirements and license terms. |

Before choosing a model, use the same held-out document pages and prompts to compare Arabic transcription accuracy, extraction accuracy, valid structured outputs, inference time, and peak memory. Include small text, diacritics, stamps, and poor scans. Measure the effect of image resolution rather than assuming the current 600-pixel preprocessing setting is sufficient.

The selected checkpoint must fit local hardware and the low-budget goal. Record its exact model ID, revision, precision or quantization settings, image processing configuration, and library versions when implementing inference. The project owner has selected `google/gemma-3-4b-it` based on experiments described below. A model-loading cell is now included in the notebook; a baseline text-generation cell is now included but has not been executed or verified by the agent.

## Practical model experiments and selection

These are qualitative observations reported by the project owner from separate experiments. They are not a controlled benchmark or a guarantee about every checkpoint in a model family. Training settings, prompts, hardware, generation limits, and evaluation samples have not yet been recorded here.

| Model tested | Reported behavior | Project decision |
| --- | --- | --- |
| Qwen3-VL Thinking variant | Reasoning before the answer increased the time needed to obtain outputs. | Poor fit for the desired practical OCR workflow. The exact checkpoint ID still needs to be recorded. |
| Qwen3-VL 2B Instruct variant | Initial results appeared promising for finetuning, but after finetuning the model added words that were absent from the scanned document. | Not selected for this task because unsupported additions are unacceptable in legal transcription. The complete checkpoint ID and finetuning configuration still need to be recorded. |
| LLaVA-NeXT | Faster than Gemma 3. After finetuning, it performed well on documents with limited content, but did not produce the full long text for information-dense documents. | A useful comparison model, with a reported completeness limitation on dense pages. The cause of incomplete output has not been established; generation limits and image resolution should be checked. |
| Gemma 3, with `google/gemma-3-4b-it` chosen | Slower, but the owner's concluding assessment favored its transcription fidelity: it reproduced text well without observed invented content in those tests, although some details could be missed. | Selected for the next implementation stage. The owner also mentioned hallucination in an earlier Gemma comparison; the checkpoint and settings for that observation are unspecified, so further evaluation remains necessary. |

Selected checkpoint: [google/gemma-3-4b-it](https://huggingface.co/google/gemma-3-4b-it).

### Fidelity requirement for legal documents

Transcription must reproduce the visible source text without adding inferred words or completing unclear passages from context. Missing content and invented content must be evaluated separately. Future extraction and question-answering outputs should remain separate from verbatim transcription so generated interpretations cannot be confused with source text.

Before accepting a finetuned model, compare its output with manually checked reference transcriptions, including dense pages and poor scans. Record unsupported additions, omissions, character and word errors, output truncation, and latency. The current choice of Gemma is based on observed behavior; it does not establish that hallucination is impossible.

## Implementation status

The current implementation prepares scanned Arabic PDFs as images locally in VS Code. It supports single-document and batch conversion. The notebook also includes Gemma model loading and a baseline transcription generation cell, whose full execution has not been verified by the agent. Structured data generation, model finetuning, and deployment remain planned.

## Local setup

Use Python 3.11 and run these commands from the project folder:

```bash
uv venv --seed --python 3.11 .venv
source .venv/bin/activate
uv pip install -r requirements.txt
```

If Python 3.11 is already installed, you can also use standard Python tools:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

The starter dependencies cover PDF rendering, image processing, datasets, VLM inference, LoRA adapters, and Weights & Biases tracking. The virtual environment is excluded from Git.

LlamaFactory and vLLM are tools considered for future training and deployment. If a rented Linux GPU environment such as Vast.ai is needed, its use should fit the low-budget goal. GPU-specific dependencies will be selected alongside the target model and CUDA version; a remote training environment does not change the goal of local model inference.

## Hugging Face authentication

Create a read-only or appropriately scoped token in [Hugging Face token settings](https://huggingface.co/settings/tokens). Revoke any token exposed in a chat or committed file. Model downloads normally require read access, not write permissions.

For reuse across local projects, activate the environment and run:

```bash
source .venv/bin/activate
hf auth login
```

Enter the replacement token at the interactive prompt. Hugging Face stores it in its local authentication cache, which compatible libraries can use across projects. Adding it to Git credentials is unnecessary for model downloads. Access to Gemma may also require accepting its terms on the model page with the same Hugging Face account.

A local `.env` file is provided with an empty `HF_TOKEN` field. Set your replacement token there locally and load it before model access. For a fresh clone, copy `.env.example` to `.env`:

```python
from dotenv import load_dotenv

load_dotenv(project_dir / ".env")
```

`.env` is ignored by Git; `.env.example` contains no credential. Do not print tokens or save them in notebook cells or outputs. An environment token can override cached authentication, so remove an obsolete `HF_TOKEN` when switching accounts or tokens. Authentication does not automatically download a model or enable inference.

The notebook includes a **Hugging Face authentication (local)** section after the imports and project-path setup. It reads `HF_TOKEN` directly from `.env` with `dotenv_values`, validates it through `HfApi.whoami`, and calls `login(token=hf_token, add_to_git_credential=False)` to cache authentication locally. Its output shows only the account name. Missing or invalid credentials raise a message without printing the token. This cell can be skipped for PDF-only work and does not load Gemma. Dependencies: `python-dotenv` and `huggingface-hub`, both listed in `requirements.txt`. The login saves a credential in the user’s Hugging Face cache, outside the repository.

## VS Code notebooks

Install the Python and Jupyter extensions in VS Code, open this project folder, and open `notebooks/ocr-finetune-vlm.ipynb`. Use **Select Kernel → Python Environments** and select `.venv/bin/python`. The notebook prints the selected interpreter so you can verify it.

PDF files are loaded from the local `data/` folder. The notebook renders PDF pages with PyMuPDF, which is installed through `requirements.txt`; no external Poppler installation is required.

If an import fails, check the interpreter printed by the notebook. Its dependency installation cell uses `%pip` to install PyMuPDF, Pillow, and tqdm into the active kernel. Restart the kernel after installation and run the cells in order.

## Project structure

```text
.
├── README.md
├── AGENTS.md
├── requirements.txt
├── data/
│   ├── README.md
│   └── *.pdf
├── notebooks/
│   └── ocr-finetune-vlm.ipynb
└── outputs/
    └── images/
        └── <pdf_name>/
            ├── page_001.jpg
            └── ...
```

`data/README.md` maps descriptive PDF filenames to their Arabic titles and original filenames. The local `.venv/` and generated `outputs/` directories are excluded from Git.

## PDF preprocessing pipeline

Run the notebook from top to bottom. It resolves `data/` relative to the project root, supporting execution from either the root or the `notebooks/` directory.

1. **Environment and imports:** print the Python interpreter, load libraries, and locate PDF inputs. Python's built-in `glob` finds files matching `*.pdf`; no additional package is needed for it.
2. **Image preprocessing:** `preprocess_image(image, max_width=600)` converts a Pillow image to grayscale, reduces its width when necessary using Lanczos resampling, preserves its aspect ratio, and increases contrast by a factor of 1.5. Smaller images are not enlarged.
3. **PDF conversion:** `convert_pdf_to_images(pdf_path, output_base_dir, max_width=600)` uses PyMuPDF to render one page at a time at 200 DPI in RGB, preprocesses it, and saves an optimized grayscale JPEG with quality 85. It returns the list of generated image paths.
4. **Single-document test:** convert `data/reconciliation_center_regulations.pdf`, save its pages under `outputs/images/reconciliation_center_regulations/`, and print the result count. Change `sample_pdf_path` to test another document.
5. **Batch conversion:** list all PDFs directly inside `data/`, sort their paths, and convert them using a tqdm progress bar. `all_generated_paths` maps each PDF filename to its generated page paths; the final message reports the total number of documents and images.

Example after running the import and function cells:

```python
generated_paths = convert_pdf_to_images(
    pdf_path=str(Path(data_dir) / "reconciliation_center_regulations.pdf"),
    output_base_dir=str(project_dir / "outputs" / "images"),
    max_width=600,
)
```

Output pages use one-based numbering: `page_001.jpg`, `page_002.jpg`, and so on. Re-running conversion overwrites files with matching names. Existing extra files are not automatically removed. Batch discovery searches the top level of `data/` only and stops if a document raises an error.

### Image quality considerations

The width of 600 pixels is a configurable starting point, not a validated optimum for Arabic OCR. Small text and diacritics may require larger images. Grayscale conversion, contrast adjustment, and JPEG compression can change or remove useful detail; compare processed images with the original scans before preparing a training dataset. This pipeline prepares images but does not measure OCR accuracy.

## Dependencies and their roles

| Dependency | Purpose | Current use |
| --- | --- | --- |
| PyMuPDF | Render PDF pages without external system tools | PDF conversion |
| Pillow | Image conversion, resizing, contrast, and JPEG export | Image preprocessing |
| tqdm | Progress reporting | Batch conversion |
| ipykernel | Execute notebooks in a Python environment | Local VS Code kernel |
| pdf2image | Alternative PDF rendering wrapper requiring Poppler | Retained dependency; not used by the current notebook |
| datasets, pydantic, python-dotenv | Dataset management, structured data validation, and environment configuration | Available for future work |
| torch, transformers, accelerate, peft | Model execution, training support, and LoRA adapters | Available for future work |
| wandb | Experiment tracking | Available for future work |

Version constraints are maintained in `requirements.txt`. They specify compatible ranges rather than a complete environment lock.

## Verification

The single-document pipeline was executed locally on the four-page Reconciliation Center PDF. All four outputs were verified as readable grayscale JPEGs with widths at most 600 pixels. The batch conversion cell has passed Python syntax checks; full batch execution has not yet been verified.

## Additional document extraction tasks

[DOCUMENT_EXTRACTION_TASKS.md](DOCUMENT_EXTRACTION_TASKS.md) describes how to extend document processing beyond transcription: classification, source metadata and dates, image properties, seals, barcodes/QR codes, signatories, approval chains, routing, full text, tables, lists, and charts. It consolidates the user-provided reference screenshots into field tables with proposed types, practical uses, and source-fidelity rules.

This is a design reference, not an implemented extraction pipeline. It separates verbatim transcription from classification, visual descriptions, and explicitly requested translations. Future extraction should use targeted prompts, preserve original Arabic values, represent missing or uncertain fields explicitly, and retain page-level evidence. Structured JSON validation and task-specific evaluation remain to be implemented.

The reference also includes a proposed JSON envelope, explicit field statuses and evidence, targeted prompts, handling of dense pages, validation checks, evaluation metrics, and execution traceability. These additions guide future development; they do not implement new extraction tasks in the notebook.

## Documentation maintenance

Every new or changed feature must be documented in this README with its purpose, usage, dependencies, inputs, outputs, and limitations. Implemented functionality and planned capabilities must remain clearly identified.

## Base-model evaluation setup

The notebook’s **Evaluate Base Model** section prepares `google/gemma-3-4b-it`, the selected instruction-tuned checkpoint, rather than the pretrained `google/gemma-3-4b-pt` variant shown in the reference screenshot. It imports `AutoProcessor` and `Gemma3ForConditionalGeneration` from Transformers and uses PyTorch to detect CUDA, Apple MPS, or CPU. Hardware detection alone does not establish that the full model fits in memory or runs successfully.

The input is `outputs/images/reconciliation_center_regulations/page_001.jpg`, created by the single-PDF conversion cell. Pillow opens it and converts it to RGB as `sample_image`. The cell defines `model_id`, `sample_image_path`, and `device`, and reports the image dimensions. No HTTP image-download library is required for this local input.

Run the imports and project-path setup, then the single-PDF conversion, before this section. This stage checks imports, device availability, and image preparation only: it does not download model weights, instantiate a processor or model, generate text, or calculate evaluation metrics. A subsequent cell now loads the model and processor; a baseline transcription cell follows, while scored evaluation metrics remain to be implemented. Dependencies are already included in `requirements.txt`: Transformers, PyTorch, and Pillow.

### Missing model dependencies in VS Code

If the evaluation imports raise `ModuleNotFoundError` for `transformers` or `torch`, the selected notebook kernel may differ from the project environment. Select `.venv/bin/python`, or run the **Install model dependencies** cell immediately before the evaluation imports. It installs Transformers, PyTorch, Accelerate, and Pillow into the active kernel using `%pip`. Restart the kernel afterward and rerun the required setup cells. Model weights are not downloaded by this installation cell.

### ChromaDB conflict in a shared environment

The shared `ai-agent` environment was reported to contain `chromadb==0.5.23`, which requires `tokenizers<=0.20.3`, while Transformers 4.57 requires `tokenizers>=0.22.0`. These requirements cannot be satisfied together. Do not downgrade tokenizers in the VLM environment to address this conflict. Use the dedicated project `.venv`, which does not require ChromaDB, and keep unrelated projects in separate environments.

In VS Code, use **Select Kernel → Python Environments** and choose this project's `.venv/bin/python`. If it is not listed, use **Python: Select Interpreter → Enter interpreter path** from the Command Palette, select `.venv/bin/python` from this project, and then select that environment as the notebook kernel. Restart the kernel and rerun the setup cells. Verify the interpreter printed by the first code cell points to this project's `.venv`, rather than `miniconda3/envs/ai-agent`.

### Sample image preview

After the base-model setup cell, **Preview the sample page** displays `sample_image_path` inline using `IPython.display.Image` (aliased as `IPImage` to avoid a naming conflict with Pillow). The path is converted to a string for the display API. This provides a visual check of the processed page before inference; it does not extract text. IPython is supplied by the project’s `ipykernel` dependency.

### Load Gemma weights and processor

After authentication and base-model setup, the notebook calls `Gemma3ForConditionalGeneration.from_pretrained(model_id, dtype="auto", device_map="auto").eval()` and `AutoProcessor.from_pretrained(model_id)`. First use downloads weights and processor files into the Hugging Face cache outside the repository. The account must have access to Gemma. Dependencies: Transformers, PyTorch, Accelerate, and Hugging Face Hub.

The outputs are `model` and `processor`; the cell prints the checkpoint, floating-point dtype, and device map. Automatic dtype preserves checkpoint precision and is not quantization. Automatic placement depends on memory and hardware and may differ from the previously detected `device` variable. Inspect the printed device map before generation. Weights require several GB and inference consumes additional memory; this setup is not a guarantee that the model fits local hardware. `.eval()` sets evaluation behavior but does not disable gradient recording; future inference should use `torch.inference_mode()`.

This loading cell has passed Python syntax checks. Full weight loading has not been executed by the agent, and no transcription or accuracy result is established by this change.

### Baseline transcription generation

After loading `model` and `processor`, the notebook sends the RGB `sample_image` and a transcription prompt through `processor.apply_chat_template`. Inputs are placed on `model.device`; image tensors are cast to the model dtype while token IDs remain integers. For an automatically distributed or offloaded model, check the printed device map and runtime placement before assuming this single-device input setup is appropriate.

The prompt requests exact visible Arabic text, preserved numbers and reading order, no inferred content, and `[غير مقروء]` for unreadable spans. Markdown should reflect source structure. Generation uses `torch.inference_mode()`, greedy decoding (`do_sample=False`), and `max_new_tokens=1024`. Prompt tokens are excluded from decoding. The output variable is `decoded`; the cell prints it with generation duration and token count. Timing excludes model loading and preprocessing. A warning signals that the token budget was reached; increase it if necessary for dense pages and assess the time/memory cost.

Greedy decoding and prompt instructions do not guarantee accuracy or prevent hallucination. Compare additions, omissions, and numbers against the original scan. This code has passed syntax checks, but full model generation and fidelity have not been verified by the agent. No scored OCR benchmark or automatic transcription export is implemented yet.
