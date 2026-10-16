# Full-text systematic-review decision engine

A reproducible Python workflow for a blockchain-based land-registry systematic literature review.

## What it does

- Links local PDFs to the literature dataset using `RefNo`.
- Treats `Source` as the article title.
- Extracts text page-by-page using PyMuPDF and stores page-level text.
- Calculates SHA-256 hashes for PDFs and extracted text.
- Optionally uses an LLM to produce **provisional** eligibility assessments and RQ1-RQ4 evidence candidates.
- Validates every AI-supplied quotation against the extracted PDF page text before retaining it.
- Never silently treats uncertain criteria as an exclusion.
- Produces a single-reviewer review queue, quality appraisal sheet, evidence matrix, performance metrics queue, data-quality report, and append-only audit log.
- Exports CSV files and an Excel workbook.

AI output is decision support, not a substitute for full-text assessment. A human reviewer must verify eligibility, extracted claims, quotations, quality appraisal and final decisions.

## Folder layout

```text
land_registry_review_engine/
├── .env.example
├── requirements.txt
├── config/review_protocol.json
├── data/literature_dataset.xlsx
├── full_texts/REF001.pdf
├── src/review_engine.py
└── results/
```

PDF filenames should match `RefNo`, for example `REF001.pdf`. The engine also supports a `--pdf-dir` folder containing PDFs named this way.

## 1. Install

Python 3.10+ recommended.

### Windows PowerShell

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
Copy-Item .env.example .env
```

### Linux / macOS / GitHub Codespaces

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
cp .env.example .env
```

## 2. Prepare the data

Place your Excel/CSV metadata file at `data/literature_dataset.xlsx`. The dataset should use your established columns. The engine requires `RefNo` and `Source`; it will preserve any other metadata columns.

Place article PDFs in `full_texts/` using the `RefNo` as the filename:

```text
full_texts/
├── REF001.pdf
├── REF002.pdf
└── REF003.pdf
```

Do not rename records to a guessed reference. Check `RefNo` carefully. Papers that are not available should remain in the dataset and be reported as `Full text unavailable`; they are not automatically excluded.

## 3. Run without AI

This creates the review queue and extracts PDF text. It does not need an API key.

```bash
python src/review_engine.py --input data/literature_dataset.xlsx --pdf-dir full_texts --output-dir results
```

CSV example:

```bash
python src/review_engine.py --input data/literature_dataset.csv --pdf-dir full_texts --output-dir results
```

## 4. Run with AI-assisted extraction

1. Open `.env` and set `OPENAI_API_KEY` to your own API key.
2. Confirm that the model name in `OPENAI_MODEL` is available to your account.
3. Run:

```bash
python src/review_engine.py --input data/literature_dataset.xlsx --pdf-dir full_texts --output-dir results --ai
```

The engine sends extracted text excerpts to the configured model. Review your institution's data-handling requirements before sending unpublished, licensed, confidential, or otherwise restricted material to an external AI service. Do not share API keys or commit `.env` to version control.

For large papers, the engine processes page batches to limit prompt size. AI outputs remain provisional and every quote must match the source text.

## 5. Output files

- `review_engine.xlsx`: workbook with overview, full-text inventory, eligibility queue, RQ evidence, performance evidence, quality appraisal, and data quality.
- `full_text_pages.csv`: page-level extracted text.
- `eligibility_assessments.csv`: provisional criterion-level assessments and reviewer decision fields.
- `rq_evidence.csv`: RQ1-RQ4 evidence candidates with `RefNo`, title, page, quote, rationale and verification status.
- `performance_evidence.csv`: candidate performance metrics and source page.
- `quality_appraisal.csv`: appraisal checklist with reviewer fields.
- `data_quality_issues.csv`: missing metadata, missing PDFs, duplicate identifiers, unreadable PDFs and other issues.
- `audit_log.jsonl`: append-only JSON Lines event log.
- `run_manifest.json`: protocol version, timestamps, input file hash, PDF hashes, configuration and run options.

## 6. Human review workflow

Open `review_engine.xlsx` and review each row.

### Eligibility
- `AI Criterion Assessment` is a candidate only.
- Set `Reviewer Criterion Decision` to `Yes`, `No`, or `Unclear`.
- Set `Final Eligibility Decision` to `Include`, `Exclude`, or `Needs human review`.
- Every exclusion must have a specific, protocol-based reason.
- Full-text unavailability is tracked separately and does not equal topic-based exclusion.

### Evidence
- Check the exact quote and page against the PDF.
- Change `Verification Status` to `Verified`, `Rejected`, or `Needs review`.
- Record your interpretation and notes.
- Do not use a quote if the page or quote validation fails.

### Quality appraisal
- Set each criterion to `Yes`, `Partial`, `No`, or `Unclear`.
- Use notes to justify assessments. Do not use a single arbitrary quality score to automatically exclude a study unless that rule is in your registered protocol.


## Alternative mode: one consolidated synthesis PDF

If you have a single summarized research PDF containing synthesis claims, citations to your study IDs, and tables, you do **not** need to provide every original article PDF to run this mode. Keep your CSV/XLSX metadata file with `RefNo` and `Source` (where `Source` is the article title), then run:

```bash
python src/review_engine.py --input data/literature_dataset.xlsx --summary-pdf data/research_summary.pdf --output-dir results
```

CSV equivalent:

```bash
python src/review_engine.py --input data/literature_dataset.csv --summary-pdf data/research_summary.pdf --output-dir results
```

For AI-assisted claim extraction and RQ classification, configure `OPENAI_API_KEY` in `.env` and run:

```bash
python src/review_engine.py --input data/literature_dataset.xlsx --summary-pdf data/research_summary.pdf --output-dir results --ai
```

### Summary-PDF outputs

- `summary_review.xlsx`: workbook with summary claims, citation links, reference coverage, extracted tables, page text, unmatched citation-like numbers, and reviewer quality-appraisal fields.
- `summary_claims.csv`: one row per synthesis-claim candidate, with the summary PDF page and cited `RefNo` values.
- `citation_links.csv`: one row per claim-to-reference relationship, joined to `Source` (article title) and DOI metadata.
- `reference_coverage.csv`: shows which dataset references appear in the consolidated PDF.
- `summary_pdf_tables.csv`: best-effort table extraction with page and row locations; visually verify table structure against the PDF.
- `summary_pdf_pages.csv`: extracted text by page.
- `unmatched_citation_numbers.csv`: possible citation-like numbers that did not map cleanly to a `RefNo`; review before treating them as citations.
- `summary_pdf_inventory.csv`, `quality_appraisal.csv`, `run_manifest.json`, and `audit_log.jsonl`.

The rule-based mode finds sentences that contain known `RefNo` values and assigns RQ candidates using transparent keyword patterns. `--ai` can structure claims and classify RQ1–RQ4, but outputs remain provisional. A cited synthesis statement such as “... improve commit times 578 763 769 770” is stored as one synthesis claim linked to four study records—not as four independently verified quotations from the original papers. Citation numbers are matched to the exact `RefNo` field; `Source` remains the article title.

**Methodological limitation:** this mode analyzes the consolidated summary, not the full text of each cited study. It can support synthesis extraction and traceability, but it cannot independently verify the original studies' methods, results, eligibility, or quality unless those details are adequately represented in the summary. Table extraction is best-effort and must be checked against the PDF, especially for complex or scanned tables. Keep page-level evidence and manually verify claims and citation links.

## Protocol changes

Edit `config/review_protocol.json` before screening. Record substantive changes to eligibility rules with a new protocol version. Rerun the engine after changes; keep prior exports if you need a complete historical record.

## Important limitations

- The PDF text extractor does not reliably interpret every table, figure, formula or scanned page. Scanned PDFs may require OCR; inspect extraction quality.
- AI can misclassify evidence or misunderstand context. Quote validation catches unsupported quotations but does not prove the interpretation is correct.
- Performance values require human verification of units, test setup, workload, network, and whether the value is measured, simulated, or proposed.
- This tool supports transparent screening and evidence synthesis; it does not independently certify a review as PRISMA-compliant or replace reviewer judgement.
