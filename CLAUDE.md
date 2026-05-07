# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project purpose

Research repository for papers/presentations on using LLMs to do **discourse analysis**, with or in lieu of corpus tools. The current experiment probes how different LLMs interpret the word "use" in 18 U.S.C. § 924(c)(1) when asked to apply it to the facts of *Smith v. United States* — i.e., whether trading a firearm for drugs counts as "using" the firearm. The notebooks generate ~100 completions per model and the analysis notebook compares them.

## Environment

- Conda environment: `llms_da` (Python 3.11). Definition in `environment.yml`.
- Activate: `conda activate llms_da`. Recreate from scratch: `conda env create -f environment.yml`.
- All notebooks are run from the `code/` directory and use `os.path.join('..', 'data')` to reach the corpus — preserve that working directory when running cells.

## Repository layout

- `code/` — Jupyter notebooks, one per model/SDK/round. Naming convention: `smith_<model>[_<variant>].ipynb`. The `_3` suffix marks the third (2026-05) round of generations against the latest models.
  - **Round 1–2 (2024–25):**
    - `smith_chat_gpt.ipynb`, `smith_chat_gpt_2.ipynb` — OpenAI (`gpt-4o`)
    - `smith_claude.ipynb` — Anthropic (`claude-3-5-sonnet-20240620`)
    - `simth_gemini.ipynb`, `smith_gemini_2_sdk.ipynb`, `smith_gemini_2_api.ipynb` — Google Gemini, two different SDKs (`google-genai` vs. legacy `google-generativeai`) and different temperatures
    - `smith_mistral.ipynb` — Mistral
  - **Round 3 (2026-05):**
    - `smith_claude_3.ipynb` — Anthropic (`claude-opus-4-7`)
    - `smith_chat_gpt_3.ipynb` — OpenAI (`gpt-5.5`)
    - `smith_gemini_3.ipynb` — Google Gemini (`gemini-3.1-flash-lite-preview`, via `google-genai` SDK)
  - `analyze_responses.ipynb` — cross-model sampling/comparison; writes `data/sample_responses.md`
- `data/` — inputs and outputs of the experiment.
  - Inputs: `18USC924c1.txt` (statute), `smith_case_summary.txt` (facts).
  - Outputs: `smith_responses_<model>[_<variant>].csv` with columns `answer`, `reasoning` (one row per generation, ~100 rows per file). Round-3 CSVs encode the model family in the filename: `smith_responses_claude_opus_4_7.csv`, `smith_responses_gpt_5_5.csv`, `smith_responses_gemini_3_1_flash_lite.csv`.

## Credentials

Two patterns coexist; pick the one that matches the round.

- **Round-3 notebooks (`smith_*_3.ipynb`):** API keys live in a git-ignored `.env` file at the repo root and are loaded via `python-dotenv`. The provider SDKs read them from the environment automatically (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GEMINI_API_KEY`) — no helper, no `api_key=` argument. Required entries in `.env`:
  ```
  ANTHROPIC_API_KEY=sk-ant-...
  OPENAI_API_KEY=sk-...
  GEMINI_API_KEY=...
  ```
- **Round 1–2 notebooks (legacy):** `code/credentials.py` is git-ignored and provides per-provider helpers (`get_credentials_openai()`, `get_credentials_gemini()`, `get_credentials_claude()`, `get_credentials_mistralai()`). The old notebooks import these directly. The file stays in place to keep the legacy notebooks runnable; do not retrofit them.

**The committed `.gitignore` excludes `credentials.py`, `.env`, `__pycache__/`, and `.ipynb_checkpoints/`. Do not commit notebooks, logs, or CSVs that contain raw keys, and do not paste keys into notebook cells.**

## Notebook conventions

Every model notebook follows the same shape — replicate it when adding a new provider so `analyze_responses.ipynb` keeps working:

1. Import provider SDK + load credentials. Round-3 notebooks: `from dotenv import load_dotenv; load_dotenv(os.path.join('..', '.env'))` and instantiate the client with no arguments. Round 1–2 notebooks: `from credentials import get_credentials_*`.
2. Define `get_completion(prompt: str) -> str` wrapping the provider's chat/completion call.
3. Read `statute` and `case_summary` from `../data/`.
4. Build two prompt variants:
   - `prompt` — asks the model to *show* its analysis inside `<legal_interpretation>` tags.
   - `new_prompt` — same instructions, but the analytical steps are silent; the response is constrained to `ANSWER: <yes/no>` followed by `REASONING: <text>`. **The 100-run loop always uses `new_prompt`.**
5. Loop 100 times, parse with `find("ANSWER:")` / `find("REASONING:")` slicing, build a DataFrame with `answer`/`reasoning` columns, save to `../data/smith_responses_<model>.csv`.

Round-3 notebooks add a **probe cell** between client setup and `get_completion`: it issues a one-token call and prints `response.model` (or `response.model_version` for Gemini). This pins the exact dated snapshot the API resolves the family alias to into the saved notebook output, so the precise micro-version used for the 100-call run is preserved alongside the data.

When adding a new model, keep the prompt text identical (including the case-sensitive `ANSWER:` / `REASONING:` markers) so the parser and the cross-model analysis sample the same construct. If the model wraps the answer in markdown (e.g. `Yes**\n\n**`), normalize it with a `.loc[...]` fix-up before saving — see `smith_chat_gpt_2.ipynb` cell 13 for the established pattern.

## Running a notebook

From `code/`:
```
jupyter lab        # or: jupyter notebook
```
Then execute cells top-to-bottom. The 100-call loop hits the provider's API directly with no rate-limit handling, so expect minutes of wall time and watch for quota errors.

## Provider/SDK notes

### Round 1–2 (legacy)
- Gemini has two notebooks intentionally: one uses the new `google-genai` client (`smith_gemini_2_sdk.ipynb`, with `temperature=1.7`), the other uses the legacy `google-generativeai` (`smith_gemini_2_api.ipynb`, default temp). The CSV filenames reflect the variant (`_temp_1.7`, `_api`).
- Claude notebook pins `claude-3-5-sonnet-20240620` and sets `system="You are an ordinary native speaker of English"` — the system prompt is part of the experiment, don't drop it when iterating.
- OpenAI notebook uses `gpt-4o` via `client.chat.completions.create`.

### Round 3 (2026-05)
- **Claude (`smith_claude_3.ipynb`):** `claude-opus-4-7`, `temperature=1.0`, `max_tokens=2048`. Same `system="You are an ordinary native speaker of English"` as the legacy notebook — preserved deliberately.
- **OpenAI (`smith_chat_gpt_3.ipynb`):** `gpt-5.5` via `client.chat.completions.create`. **Temperature is intentionally not passed** — at the time of writing, the OpenAI docs for `gpt-5.5` did not confirm `temperature` as supported (recent reasoning-capable flagships often reject it), so the notebook relies on the model's default sampling. If you confirm temperature is accepted, set it explicitly.
- **Gemini (`smith_gemini_3.ipynb`):** `gemini-3.1-flash-lite-preview` via the current `google-genai` SDK (`from google import genai`; `client.models.generate_content(...)`). The legacy `google-generativeai` SDK is deprecated and is not used in this round. Switched from `gemini-3.1-pro-preview` to Flash-Lite for tractable latency on the 100-call loop. **Temperature is intentionally not passed** for the same documentation reason as GPT-5.5.

### Future work
The `analyze_responses.ipynb` notebook is the natural candidate for a **Marimo** port — its workload (filter, slice, re-render across the saved CSVs) benefits from Marimo's reactive model and would let a presentation slide through model/sample-size/variant choices live. The data-generation notebooks (`smith_*.ipynb`) intentionally stay in Jupyter: they are fire-once-save-CSV, and Marimo's reactivity would risk re-firing the paid 100-call loop on upstream changes.
