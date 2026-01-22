# AI Marksheet Extraction API (FastAPI + OCR + LLM)

Build an API that takes a marksheet image/PDF and returns a structured JSON with extracted fields **and confidence scores**.

## Features
- **Input**: JPG/PNG/PDF (max **10 MB**)
- **Output**: Consistent JSON schema with
  - Candidate details (name, parent name(s), roll, reg, DOB, exam year, board/university, institution)
  - Subject-wise marks/credits + grades (if present)
  - Overall result/grade/division
  - Issue date/place (if present)
  - **Per-field confidence** (0–1)
- **OCR**: Open-source `pytesseract` (with word-level confidences + optional bounding boxes)
- **LLM**: Used for **structuring/normalizing/validation** (OpenAI/Gemini via OpenAI-compatible API)
- **Concurrency**: Handles multiple requests concurrently (OCR runs in a threadpool)
- **Error handling**: size/format validation, bad PDFs, OCR/LLM failures
- **Bonus**: Optional API-key auth, batch endpoint

## Project Layout
- `app/main.py` — FastAPI app + routes
- `app/core/config.py` — settings via environment
- `app/core/security.py` — optional API key auth
- `app/schemas/marksheet.py` — output schema (Pydantic)
- `app/services/ocr.py` — OCR for images/PDFs + bounding boxes + OCR confidence
- `app/services/llm.py` — LLM client + prompt
- `app/services/extract.py` — orchestration (OCR → LLM → post-validate → confidence merge)

## Setup (Windows)
1) Create venv and install deps

```bash
python -m venv .venv
.\.venv\Scripts\activate
pip install -r requirements.txt
```

2) Install **Tesseract OCR**
- Windows: install from `https://github.com/UB-Mannheim/tesseract/wiki`
- Ensure `tesseract.exe` is in PATH **or** set `TESSERACT_CMD` env var.

3) (Optional) For PDF → image conversion, install **Poppler**
- Download poppler for Windows and add `bin/` to PATH, or set `POPPLER_PATH`.

4) Configure environment
- Copy `env.example` to `.env` and fill in LLM values.

```bash
copy env.example .env
```

5) Run API

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Open docs at `http://localhost:8000/docs`

## Environment Variables
- **LLM_PROVIDER**: `openai_compatible` (default), `ollama`, or `gemini`
- **LLM_BASE_URL**: e.g. `https://api.openai.com/v1` (for openai_compatible) or `http://localhost:11434` (for ollama)
- **LLM_API_KEY**: your key (OpenAI or Gemini API Key)
- **LLM_MODEL**: e.g. `gpt-4o-mini`, `llama3`, or `gemini-1.5-flash`
- **API_KEY** (optional): if set, requests must include header `X-API-Key: <value>`
- **TESSERACT_CMD** (optional): full path to `tesseract.exe`
- **POPPLER_PATH** (optional): directory containing poppler `bin` executables

## API
### `POST /v1/extract`
Form-data:
- `file`: marksheet file
- `include_bboxes` (optional bool)

### `POST /v1/extract/batch`
Form-data:
- `files`: multiple marksheet files
- `include_bboxes` (optional bool)

## Confidence Logic (summary)
Each field returns:
- `value`
- `confidence` in `[0,1]`
- `sources` (optional): OCR span evidence and LLM confidence

Confidence is derived by combining:
- **OCR confidence**: average word confidence for the text span supporting the field (when detectable)
- **LLM self-check**: LLM returns a per-field confidence after validating consistency (dates, totals, plausible ranges)
- Final score = weighted blend (default OCR 0.65, LLM 0.35), clipped to `[0,1]`.

Full details are in `APPROACH.md`.

## Notes on Deployment
Deploy with any provider (Render/Fly.io/Azure/etc). Keep secrets in platform env vars. This repo contains no credentials.


