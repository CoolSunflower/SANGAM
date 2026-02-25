# SANGAM: SystemVerilog Assertion Generation via Monte Carlo Tree Self-Refine

> Official code implementation for the paper *"SANGAM: SystemVerilog Assertion Generation via Monte Carlo Tree Self-Refine"*.

## Abstract

Recent advancements in the field of reasoning using Large Language Models (LLMs) have created new possibilities for more complex and automatic Hardware Assertion Generation techniques. This paper introduces **SANGAM**, a SystemVerilog Assertion Generation framework using LLM-guided Monte Carlo Tree Search for the automatic generation of SVAs from industry-level specifications. The proposed framework utilizes a three-stage approach:

1. **Stage 1 — Multimodal Specification Processing**: Signal Mapper, SPEC Analyzer, and Waveform Analyzer LLM Agents parse the design specification PDF and extract per-signal information.
2. **Stage 2 — Monte Carlo Tree Self-Refine (MCTSr)**: An MCTS-based reasoning loop that iteratively generates, critiques, and refines SVA candidates for each signal, guided by formal verification feedback from JasperGold.
3. **Stage 3 — Assertion Combination & Post-Processing**: MCTSr-generated reasoning traces are combined, syntax-corrected via JasperGold, and deduplicated to produce the final assertion set.

Our framework generates robust sets of SVAs, outperforming recent methods in evaluation.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Preparing a Design](#preparing-a-design)
- [Running the Pipeline](#running-the-pipeline)
- [Output Files](#output-files)
- [Analysis](#analysis)
- [Supported Designs](#supported-designs)
- [Troubleshooting](#troubleshooting)

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    SANGAM Pipeline                           │
│                                                              │
│  ┌──────────────┐    ┌────────────────┐   ┌────────────────┐ │
│  │  Stage 1     │    │  Stage 2       │   │  Stage 3       │ │
│  │  Spec        │──▶│  MCTSr Loop    │──▶│  Combine &     │ │
│  │  Processing  │    │  (4 iters)     │   │  Post-Process  │ │
│  └──────────────┘    └───────┬────────┘   └────────────────┘ │
│        │                     │                     │         │
│   PDF → OCR text       ┌─────▼──────┐          Final SVAs    │
│   CSV signal info      │ Per iter:  │          per signal    │
│   RAG book index       │ Generate   │                        │
│                        │ Critique   │                        │
│                        │ JG Check   │                        │
│                        │ Refine     │                        │
│                        └────────────┘                        │
│                                                              │
│  LLM: DeepSeek-R1 (via Nebius AI Studio)                     │
│  Formal Verifier: Cadence JasperGold (remote via SSH)        │
│  RAG: FAISS + OpenAI Embeddings                              │
└──────────────────────────────────────────────────────────────┘
```

**Per-signal MCTSr loop (Stage 2):**
1. **Generate Weak Answer** — LLM produces initial SVA set
2. **Sample Reward** — JasperGold verifies SVAs; LLM critic scores them (−100 to +100)
3. **Node Selection** — UCB1 (C=1.4) selects the best node to expand
4. **Node Expansion** — Critic provides feedback; generator refines SVAs using JG logs + RAG context
5. **Evaluation & Backpropagation** — Updated rewards propagate up the tree
6. Repeat for `max_iter=4` iterations (early exit if all assertions pass JG)

---

## Repository Structure

```
SANGAM/
├── MAIN(i2c).ipynb           # Main pipeline notebook for I2C Master design
├── MAIN(rv).ipynb            # Main pipeline notebook for RV Timer design
├── Anaysis(i2c).ipynb        # Post-run analysis for I2C results
├── Anaysis(rv).ipynb         # Post-run analysis for RV Timer results
├── main.py                   # Standalone script version of the pipeline
├── config.py                 # Model name, system prompts, SV module templates
├── utils.py                  # PDF OCR, JasperGold SSH runner, assertion extraction
├── commands.txt              # Git clone command
├── .env                      # API keys & SSH credentials (create from template below)
│
├── datasets/
│   ├── info/                 # Per-design CSVs (signal names + spec excerpts)
│   │   ├── i2c.csv
│   │   ├── rv.csv
│   │   └── hpdmc.csv
│   ├── spec/                 # Design specification PDFs
│   │   ├── i2c.pdf
│   │   ├── rv.pdf
│   │   └── hpdmc.pdf
│   └── rtl/                  # RTL source files (Verilog) per design
│       ├── i2c/              # i2c_master_top.v, i2c_master_bit_ctrl.v, ...
│       ├── hpdmc/            # hpdmc.v, hpdmc_busif.v, ...
│       └── ...               # 20 design directories total
│
├── books/                    # Reference books for RAG
│   ├── faiss_index/          # Pre-built FAISS vector index
│   │   └── index.faiss
│   └── documents.pkl         # Serialized document chunks
│
├── LLMOutput/                # Raw LLM outputs per signal per MCTS iteration
│   ├── i2c/
│   └── rv/
│
├── TempOutput/               # Intermediate CSVs during pipeline execution
│   ├── i2c/
│   └── rv/
│
├── FinalOutputs/             # Final assertion files per signal
│   ├── i2c/
│   │   ├── {signal}_Duplicates.txt       # All syntax-valid assertions (pre-dedup)
│   │   └── {signal}_NoDuplicatesLLM.txt  # Final deduplicated assertions
│   └── rv/
│
├── JasperGold_Logs/          # JasperGold verification logs
│   ├── i2c/
│   └── rv/
│
├── output/                   # MCTS trace JSONs and result CSVs
│   └── json/
│
├── i2c/                      # Per-signal output text files + analysis CSVs for I2C
│   ├── {signal}.txt
│   ├── i2cAnalysis.csv
│   └── i2cFinal.csv
│
└── rv/                       # Per-signal output text files + analysis CSVs for RV
    ├── {signal}.txt
    ├── rvAnalysis.csv
    └── rvFinal.csv
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| **Python** | 3.9+ |
| **Tesseract OCR** | Required for PDF specification extraction. [Install guide](https://github.com/tesseract-ocr/tesseract#installing-tesseract) |
| **JasperGold** | Cadence JasperGold formal verification tool, accessible over SSH on a remote server |
| **LLM API Access** | Nebius AI Studio API keys (for DeepSeek-R1) |
| **OpenAI API Key** | Required for RAG embeddings (FAISS index building) and the optional combiner step |
| **SSH Access** | Credentials for a remote server with JasperGold + design TCL scripts set up |

---

## Installation

1. **Clone the repository:**
   ```bash
   git clone git@github.com:CoolSunflower/SANGAM.git
   cd SANGAM
   ```

2. **Create and activate a virtual environment (recommended):**
   ```bash
   python -m venv venv
   # Windows
   venv\Scripts\activate
   # Linux/Mac
   source venv/bin/activate
   ```

3. **Install Python dependencies:**
   ```bash
   pip install numpy pandas openai python-dotenv retry paramiko PyMuPDF pytesseract Pillow langchain langchain-openai langchain-community faiss-cpu tabulate unstructured pickle5 ipykernel
   ```

4. **Install Tesseract OCR:**
   - **Windows:** Download the installer from [UB Mannheim](https://github.com/UB-Mannheim/tesseract/wiki) and note the install path.
   - **Linux:** `sudo apt install tesseract-ocr`
   - **macOS:** `brew install tesseract`

   Update the Tesseract path in the notebook if needed:
   ```python
   pytesseract.pytesseract.tesseract_cmd = r'C:\Program Files\Tesseract-OCR\tesseract.exe'
   ```

---

## Configuration

### 1. Create a `.env` file

Create a `.env` file in the `SANGAM/` directory:

```env
# LLM API Keys (Nebius AI Studio - supports up to 24 keys for round-robin)
NEBIUS_API_KEY1=your_nebius_api_key_1
NEBIUS_API_KEY2=your_nebius_api_key_2
# ... add more keys if needed (NEBIUS_API_KEY3 through NEBIUS_API_KEY24)

# OpenAI API Key (used for RAG embeddings and combiner step)
API_KEY=your_openai_api_key

# JasperGold Remote Server SSH Credentials
HOST_NAME=your_jaspergold_server_hostname
USERNAME=your_ssh_username
PASSWORD=your_ssh_password
```

> **Note**: Multiple Nebius API keys enable round-robin rotation to avoid rate limits. At minimum, provide `NEBIUS_API_KEY1`.

### 2. JasperGold Server Setup

The pipeline SSH-es into a remote server to run JasperGold. You need to:

1. Ensure the remote server has Cadence JasperGold installed and licensed.
2. Place the RTL files and JasperGold TCL scripts on the remote server. The default root path is:
   ```
   /storage/ckarfa/hpdmc/
   ```
3. For each design, create a `.tcl` script (e.g., `i2c.tcl`, `rv.tcl`) that configures JasperGold to:
   - Load the RTL files
   - Bind the SVA module
   - Run formal property checking
4. The SVA file is uploaded to `sva/{design_name}/{design_name}.sv` on the remote server.

To adapt for your server, modify the `run_jg()` function in `utils.py`:
- Update `ROOT` to your remote directory path
- Update the `JG_COMMAND` shell commands (CAD tool sourcing, paths, etc.)

### 3. Model Configuration

Edit `config.py` to change the LLM model:

```python
MODEL_NAME = "deepseek-ai/DeepSeek-R1"  # Default: DeepSeek-R1 via Nebius
```

---

## Preparing a Design

To add a new design to the framework:

1. **Add the specification PDF** to `datasets/spec/{design_name}.pdf`

2. **Add the RTL files** to `datasets/rtl/{design_name}/`

3. **Create a signal CSV** at `datasets/info/{design_name}.csv` with columns:
   | Column | Description |
   |---|---|
   | `spec_name` | Design name (e.g., `i2c`) |
   | `signal_name` | Signal name (`NA` for row 0, then one signal per row) |
   | `information` | Row 0: full architecture overview + signal mapping. Rows 1+: per-signal specification excerpt |

4. **Add an SV template** in `config.py` — define the module wrapper with all signal ports and a `{properties}` placeholder:
   ```systemverilog
   module {design}_sva (
       input logic clk,
       // ... all signals ...
   );
   {properties}
   endmodule
   ```

5. **Set up JasperGold** on the remote server with a corresponding `.tcl` script and update `run_jg()` in `utils.py` with the new design's paths and commands.

---

## Running the Pipeline interactively using the Notebook

1. Open `MAIN({design_name}).ipynb` in Jupyter Notebook / VS Code.
2. Set `SPEC_NAME` in the first code cell:
   ```python
   SPEC_NAME = 'i2c'  # or 'rv', 'hpdmc'
   ```
3. Run cells sequentially. The pipeline will:
   - Extract text from the specification PDF via OCR
   - Load the signal information CSV
   - For each signal, run the full MCTSr pipeline (Stage 2)
   - Combine traces and post-process (Stage 3)
   - Save results to `FinalOutputs/{design_name}/`

> **Execution time**: Each signal takes ~15–30 minutes depending on LLM response times and JasperGold verification. A full design with 20+ signals can take several hours.

> Please note some parts of main.py are not-updated to recent version, please use the notebook for proper output.

---

## Output Files

| Path | Description |
|---|---|
| `LLMOutput/{design}/` | Raw LLM outputs: `{spec}_{signal}_{iter}_{step}.txt` where step ∈ {`WeakAnswer`, `SampleReward`, `GetHints`, `GetBetterAnswer`, `EvaluateNewNode`, ...} |
| `output/json/{spec}_{signal}.json` | Full MCTS tree trace (answers, rewards, UCB values, parent/child relationships) |
| `JasperGold_Logs/{design}/` | Pickled JasperGold result dictionaries per signal |
| `TempOutput/{design}/` | Intermediate CSVs saved during the pipeline run |
| `FinalOutputs/{design}/{signal}_Duplicates.txt` | All syntax-valid assertions before deduplication |
| `FinalOutputs/{design}/{signal}_NoDuplicatesLLM.txt` | **Final deduplicated assertions** for each signal |
| `{design}/{design}Final.csv` | Summary CSV with per-signal assertion outputs |
| `{design}/{design}Analysis.csv` | JasperGold verification results summary |

---

## Analysis

After running the pipeline, use the analysis notebooks to evaluate results:

1. Open `Anaysis({design_name}).ipynb`
2. It reads the final output files, runs each assertion through JasperGold, and classifies them:
   - **Complete Correct** — passes JG with no counterexamples
   - **Syntax Correct, Semantics Wrong** — valid SVA syntax but JG finds a counterexample
   - **Syntax Wrong** — JG reports syntax/compilation errors
3. Results are saved to `{design}/{design}Analysis.csv`

---

## Current Designs

| Design | Spec PDF | Signals | Description |
|---|---|---|---|
| **i2c** | `i2c.pdf` | 23 | I2C Master Core (Wishbone bus interface) |
| **rv** | `rv.pdf` | 10 | RISC-V Timer Module |
| **hpdmc** | `hpdmc.pdf` | ~25 | High Performance DDR SDRAM Memory Controller |

RTL for 20 additional designs is included in `datasets/rtl/` (AES, Ethernet MAC, SHA3, UART, etc.) but CSV/spec preparation is required before running.

---

## Troubleshooting

| Issue | Solution |
|---|---|
| `TesseractNotFoundError` | Install Tesseract OCR and set the correct path in the notebook |
| `Connection refused` (SSH) | Verify `HOST_NAME`, `USERNAME`, `PASSWORD` in `.env`; check server firewall |
| `API key exhausted` / rate limit | Add more Nebius API keys to `.env` (up to `NEBIUS_API_KEY24`) |
| JasperGold timeout or stall | Check the remote server's JG license and TCL script paths |
| `No assertions found` | Verify the signal CSV has correct `information` content for the signal |
| FAISS index missing | Run the RAG setup cells in the notebook to rebuild from books in `books/` |

---

## Citation

If you use this code in your research, please cite:

```bibtex
@inproceedings{gupta2025sangam,
  title={SANGAM: SystemVerilog Assertion Generation via Monte Carlo Tree Self-Refine},
  author={Gupta, Adarsh and Mali, Bhabesh and Karfa, Chandan},
  booktitle={2025 IEEE International Conference on LLM-Aided Design (ICLAD)},
  year={2025},
  month={June},
  doi={10.1109/ICLAD65226.2025.00024}
}
```
