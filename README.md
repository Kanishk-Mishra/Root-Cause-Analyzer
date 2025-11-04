# Build Log RCA (Chunked) — Devstral/Mistral

> Root-cause analysis for **huge Android build & console logs** using a **chunked LLM pipeline** with verdict-driven control flow.  
> Works on multi‑MB logs by scanning them in **chunks**, extracting **evidence blocks**, and consolidating a final **RCA summary per file**.

---

## 🔧 Problem Statement

Android builds generate logs with **thousands to millions of lines**. Finding the **true failure cause** (vs. harmless warnings) is hard and expensive to do manually.  
This project analyzes **Build Log** and **Console Log** inputs, detects **terminal failures**, and produces a concise **Root Cause Analysis (RCA)** with **evidence blocks** and a **failure category**.

---

## 🎯 Core Outputs

For each log file, the notebook produces:

- **AI Failure Summary** (≤ 12 concise bullets)  
- **Categorized Failure Type**  
  - `REPO_SYNC_FAILURE`, `MODULE_BUILD_FAILURE`, `INFRA_FAILURE`, `TEST_FAILURE`, or `UNKNOWN`
- **Root Cause Hypothesis** (1–3 sentences)
- **Evidence Blocks** (raw log text + line ranges, **no paraphrasing**)
- Results are appended to **`summary.md`** under a per‑file heading.

---

## 🧠 Approach (How It Works)

1. **File Discovery**  
   - Scans the `./input` folder for `*.log`, `*.txt`, `*.md`, `*.json` files.

2. **Whole‑File Read & Early Filter**  
   - Loads the file **without truncation**.  
   - If the file **does not contain an `exit status` line**, it’s treated as **passed/truncated** and is skipped (message is recorded in `summary.md`).

3. **Chunking**  
   - Splits the log into **chunks of `CHUNK_SIZE` lines** (default: `1000`).  
   - Sends each chunk to the model with a prompt that asks for a structured RCA and a mandatory **`VERDICT:`** line at the end.

4. **Verdict‑Driven Control**  
   - **`VERDICT: Root cause found`** → mark as serious (after a classifier check) and continue scanning (so multiple serious candidates can be collected).  
   - **`VERDICT: Non-serious errors or warnings detected. Proceeding with the next chunk`** → ignore this chunk.  
   - **`VERDICT: Error might have got truncated, kindly combine this with the next chunk and resend`** → **merge the next chunk** with the current one and re‑query (to catch boundary‑spanning errors).

5. **Seriousness Classifier**  
   - A lightweight LLM call filters out non‑serious summaries (e.g., warnings, container pending, retries, cleanup noise).  
   - Keeps only **serious candidates** for final consolidation.

6. **Root Cause Consolidation**  
   - A final LLM pass across all **serious candidates** chooses **one primary root cause** using strict prioritization:
     - `REPO_SYNC_FAILURE` > `INFRA_FAILURE` > `MODULE_BUILD_FAILURE` > `TEST_FAILURE`  
   - Optionally lists **Other Miscellaneous Issues** (short bullets).  
   - Writes the final RCA to **`summary.md`** under `## *Log-file: <name>*`.

---

## 📁 Project Structure

```
.
├─ input/                 # Put your log files here (.log, .txt, .md, .json)
├─ main.ipynb             # Jupyter notebook (pipeline implementation)
└─ summary.md             # Output (auto-created/appended)
```

---

## 🛠️ Libraries Used & Why

- **requests** — clean, minimal HTTP client for calling the Mistral/Devstral API.  
- **glob / os / time** — standard library utilities for file discovery, IO, and retry timing.  
- **Jupyter** — run the notebook end‑to‑end, inspect intermediate results, and iterate quickly.

> _Note:_ The code uses **raw REST calls** for maximum control (headers, timeouts, retries, JSON payloads), which also makes it easy to switch models or providers.

---

## 🤖 Models & Tokens (Mistral)

- API endpoint: `https://api.mistral.ai/v1/chat/completions`  
- Recommended chat models for this task:
  - `magistral-small-latest` (fast, low cost)  
  - `magistral-medium-latest` (stronger reasoning)  
- **Context window (“Max Tokens” in docs)** is the **total** of **input + output** tokens.  
  - Example: `magistral-medium` shows **128k** → your chunk + the model’s reply **together** must fit under 128k tokens.

> The notebook currently sets the model name in the payload. You can swap it freely (see _Configuration_).

---

## ⚙️ Configuration

Inside `main.ipynb`:

- **`SUPPORTED_EXTS`** — file patterns scanned in `./input`  
- **`CHUNK_SIZE`** — lines per chunk (default: `1000`). Increase cautiously; ensure (input + output) stays within model limits.  
- **`output_path`** — output file (default: `summary.md`)  
- **`retries`/`timeout`** — robust API retries already implemented.  
- **Model Name** — set in the `payload["model"]`. Prefer `magistral-small-latest` or `magistral-medium-latest`.  
- **API URL / Key**  
  - `API_URL = "https://api.mistral.ai/v1/chat/completions"`  
  - **Do not hard‑code your key in code.** Use an env var instead:
    ```bash
    setx MISTRAL_API_KEY "sk-...your-key..."    # Windows
    export MISTRAL_API_KEY="sk-...your-key..."  # macOS/Linux
    ```
    and in the notebook:
    ```python
    import os
    API_KEY = os.getenv("MISTRAL_API_KEY")
    ```

---

## ▶️ How to Run

1. **Install Python** ≥ 3.10.  
2. **Install dependencies**  
   ```bash
   pip install -r requirements.txt
   ```
3. **Prepare inputs**  
   - Put your logs in `./input`. Supported: `.log`, `.txt`, `.md`, `.json`.
4. **Set API key** (see _Configuration_).  
5. **Open the notebook**  
   ```bash
   jupyter notebook main.ipynb
   ```
   Run all cells. The pipeline will print which file is being processed and progress per chunk.
6. **Inspect output**  
   - Check **`summary.md`** for appended RCA results per file.

---

## ✅ Result Format (example)

```
## *Log-file: build_2025-09-22.log*

AI Failure Summary
- ...
- ...

Categorized Failure Type
- MODULE_BUILD_FAILURE

Root Cause Hypothesis
- ... 1–3 sentences ...

Evidence Blocks
- [Lines 5084–5102]   # (raw, unedited log text)
  <full snippet copied verbatim>
- [Lines 1395–1402]
  <full snippet copied verbatim>

Other Miscellaneous Issues
- Short bullets of secondary items (optional)
```

The **Evidence Blocks** are verbatim raw snippets (no paraphrasing), preserving the exact error strings for auditability.

---

## 🧪 Heuristics (used by the prompts)

- **Repo sync**: `repo sync`, `fatal:`, HTTP/SSL errors, hash mismatch  
- **Module build**: `FAILED:`, `ninja: build stopped`, Gradle `Task :<module> FAILED`, compiler/type/undefined reference errors  
- **Infra**: network timeouts, disk full / no space, permission denied, missing toolchains  
- **Test**: unit/instrumentation assertion failures, `FAILURES!!!`  
- **Unknown**: none of the above with confidence

Non‑serious signals (ignored as root cause): _warnings_, container pending, retries/waits, cleanup, env overrides, symlink noise, non‑blocking “file not found”.

---

## 🛡️ Privacy & Safety

- Logs may contain internal paths or tokens—ensure you **scrub** sensitive data before sharing outputs.
- The notebook only sends the **current chunk** to the API, not the entire file, reducing exposure.

---

## 🧰 Troubleshooting

- **400 Bad Request**  
  - Use `json=payload` (not `data=`).  
  - Use valid model names: `magistral-*-latest`.  
  - Print `response.text` for API error details.

- **401 Unauthorized**  
  - Missing/invalid API key. Confirm `Authorization: Bearer <key>`.

- **422 Unprocessable Entity**  
  - Malformed JSON. Ensure you pass a dict via `json=...` and include both `model` and `messages`.

- **DNS / NameResolutionError**  
  - Corporate firewall/DNS issues. Try open network or VPN; verify the endpoint domain.

- **No `VERDICT:` in response**  
  - The prompt enforces a verdict. If missing, the chunk is skipped; adjust temperature or add a retry pass.

---

## 📜 License

Add your license of choice (e.g., Apache‑2.0, MIT) to the repository root as `LICENSE`.

---

## 🙌 Acknowledgements

Thanks to Mistral AI for the chat models. This project uses **structured prompts** and a **verdict‑driven controller** to turn very large logs into compact, auditable RCA reports.
