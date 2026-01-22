# Approach Note (1–2 pages)

## Goal
Extract structured data from diverse marksheet layouts (images/PDFs) into a consistent JSON schema with **per-field confidence**.

## Pipeline Overview
1) **Ingestion & validation**
   - Accept JPG/PNG/PDF, enforce max size (10 MB)
   - Convert PDFs to images (one image per page)

2) **OCR (open-source)**
   - Run Tesseract OCR to get:
     - full text for the page
     - word-level boxes + confidences
   - Produce:
     - `raw_text`
     - `tokens[]` = `{text, conf, bbox}`

3) **LLM structuring / normalization / validation**
   - Provide the LLM:
     - the OCR `raw_text`
     - (optionally) token list summary
     - strict JSON schema instructions
   - Ask the LLM to:
     - identify candidate fields (name, roll, reg, etc.)
     - extract subject-wise rows into normalized objects
     - infer result/division and issue info if present
     - return **per-field confidence** and short rationales
   - Pydantic validates the returned JSON (types and shape).

4) **Confidence scoring**
Each field returns a `confidence` in `[0,1]`.

### Components
- **OCR confidence**:
  - If we can associate a field to OCR evidence text, we compute the **average** of word confidences across matching words (normalized string match).
  - Tesseract confidences are typically in `[0..100]` per token; we map to `[0..1]`.
  - If the evidence is not found or the field is inferred (e.g., division), OCR confidence may be `null`.

- **LLM confidence**:
  - The LLM outputs a confidence per field after self-checking:
    - date validity (DOB / issue date)
    - plausible ranges for marks/credits
    - consistency checks (e.g., totals, pass/fail wording)
  - This confidence is bounded to `[0..1]`.

### Final confidence
We blend confidences:
\[
conf = clamp( w\_{ocr}\cdot conf\_{ocr} + w\_{llm}\cdot conf\_{llm}, 0, 1)
\]
Defaults: `w_ocr = 0.65`, `w_llm = 0.35`.

If `conf_ocr` is missing, we use `conf = conf_llm` (and vice-versa).

## Why this generalizes
- OCR extracts layout-agnostic text from any board/university template.
- LLM handles variability in labeling (“Roll”, “Index”, “Enrollment”, etc.) and table structures.
- Schema enforcement + normalization reduces output drift across templates.

## Error handling & reliability
- Clear HTTP errors for invalid file types, oversized payloads, PDF conversion issues
- Timeouts for LLM calls with meaningful fallback behavior
- Concurrent requests: OCR runs in threadpool; LLM calls are async via `httpx`

## Extensions (optional)
- Bounding boxes for key fields (via token evidence)
- Batch endpoint
- API key authentication via header `X-API-Key`
- Unit tests with sample documents



