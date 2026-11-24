# Requirements: Research Tracker (MVP)

**Status**: Draft for team review
**MVP deadline**: 2026-12-16
**Sources**: `VisionDocument.md` (V), `TechnicalDocument.md` (T), `README.md` (R), `requirement-verification-questions.md` (Q1–Q20), `requirement-clarification-questions.md` (CQ1–CQ8)

Each requirement names its source in brackets so it can be traced back to a document or answer.

## 1. Intent Analysis

| Item | Assessment |
|---|---|
| User request | "Using AI-DLC, I want to start a new project. Please read the vision document and the technical environment document, then begin the AI-DLC workflow." |
| Request type | New Project (greenfield) |
| Request clarity | Clear. The vision's open questions are now resolved through two rounds of answers |
| Scope estimate | System-wide: PDF ingestion, LLM agent, retrieval, persistence, web interface, evaluation |
| Complexity estimate | Complex: multi-step LLM agent with tool calls, three local data stores that must stay consistent, background processing |
| Requirements depth | Comprehensive (traceability to sources) |

## 2. Product Summary

Research Tracker is a locally run web app for one student working on one research question. Each uploaded paper becomes a cited paper card. An LLM agent (Qwen3 8B through Ollama) then compares the paper with the papers already saved and labels it as a new angle, an addition to an existing section, redundant with an earlier paper, or not relevant. Every label has a reason and evidence. Code, not the model, checks every quote against its cited PDF page. The tracker persists across sessions and is written to a readable `tracker.md` after every successful upload. All assessments describe only the uploaded collection, never the research field as a whole. [V §1]

## 3. Key Decisions

| Topic | Decision | Source |
|---|---|---|
| Interface | Local web page (FastAPI + Jinja2) | T §1 |
| Trackers | One tracker per installation; `tracker_id` kept in data model and endpoints for later expansion | Q8 |
| Sections per paper | One primary section; optional secondary "related sections" (display-only links), assigned at upload only | Q1, CQ1 |
| Redundant vs adds | Redundant only when there is no new population, method, condition or independent dataset | Q2 |
| Agent design | Fixed pipeline for cards; capped tool-calling loop (6–8 calls) for comparison and classification | Q3 |
| Cap reached | One final tool-free call for a label; invalid output follows failure rules | CQ4 |
| Section edits | Names fixed once created; section summaries and overview rewritten | Q4 |
| Failed quotes | One retry, then shown with a "Not verified" warning; a card with no verified quote is still saved, with a card-level note | Q5, CQ2 |
| Paper display name | Extracted title, falling back to the original upload filename | CQ3 |
| Citations | Inline expandable tags on the web page; numbered footnotes in `tracker.md` | Q6 |
| Evidence view | Cited passage plus a link that opens the stored PDF at the cited page | Q7 |
| Removing papers | Not in MVP | Q9 |
| `tracker.md` | Rewritten after every successful upload; Export downloads it | Q10 |
| Processing | Background with step-by-step progress; new uploads blocked while one is processing | Q11, CQ5 |
| Failures | Retry a failing step twice; if it still fails, save nothing | Q13 |
| Speed | No fixed target; time every step and report | Q12, CQ7 |
| Model settings | Model name, context length and thinking mode in `.env`; smaller chunks and batches by default | Q14, Q16 |
| Context size | Applied through a Modelfile in the repository | CQ8 |
| Embeddings | `nomic-embed-text` through Ollama, stored in Chroma | Q15 |
| Evaluation | Script in MVP, using its own `eval-data/` folder | Q17, CQ6 |
| Extensions | Security: off. Property-based testing: partial. Resiliency: off | Q18–Q20 |

## 4. Users and Context

- **Primary users**: undergraduate and graduate students writing literature reviews or research papers; early-stage researchers and research assistants exploring a new topic. [V §4]
- **Usage context**: one user, one research question, one laptop, no accounts, no network access beyond the laptop. [V §4, T §2]
- **Core journey**: create a tracker from a question → upload papers one at a time → read cards, labels and reasons → inspect evidence → see counts and year range → close the app and continue another day. [V §5]

## 5. Functional Requirements

### FR-1 Tracker setup

- **FR-1.1** When no tracker exists, the app shows a "create tracker" page asking for a research question. [Q8]
- **FR-1.2** Creating a tracker saves the question with an empty overview and no sections. [V §2]
- **FR-1.3** When a tracker exists, the app opens straight to it. Only one tracker is supported. [Q8]
- **FR-1.4** The data model and API paths include a tracker ID so that several trackers can be added later without a redesign. [Q8]

### FR-2 Upload and validation

- **FR-2.1** The user uploads one PDF at a time. [V §2]
- **FR-2.2** The app rejects, with a clear error message and nothing saved:
  - files that do not start with `%PDF-` [T §5]
  - files over 25 MB [T §6]
  - encrypted PDFs [T §5]
  - PDFs over 20 pages [V §2, T §5]
  - PDFs with too little extractable text (for example scanned or image-only PDFs) [V §2]
  - uploads when the tracker already holds 10 papers [V §2]

  All of these checks, including text extraction with `pypdf`, run during the upload request, before any background work starts. A rejected upload gets its error immediately on the upload form, as in the endpoint example in T §6. [T §6, text-check timing decision]
- **FR-2.3** Identical PDFs are detected by SHA-256 hash before any model call and rejected with a "This PDF is already in the tracker" message. Detecting different files or versions of the same paper is out of scope. [V §2]
- **FR-2.4** Stored PDFs are named by their SHA-256 hash, never by the user's filename. The original filename is saved in the database for display only. [T §5, CQ3]
- **FR-2.5** While a paper is processing, the server rejects any new upload with a message such as "Wait until the current paper finishes." The page also disables uploading, but the server check is the one that counts. [CQ5]

### FR-3 Text extraction and chunking

- **FR-3.1** Text is extracted with `pypdf`, page by page, keeping 1-based PDF page numbers. [T §1, V §2]
- **FR-3.2** Extracted text is split into chunks. Each chunk records the PDF page or pages it came from. [V §2]
- **FR-3.3** Default chunk and batch sizes are kept small so model calls stay short and memory use stays low. Exact sizes are decided in design and measured on the test set. [Q14]
- **FR-3.4** Each chunk is embedded with `nomic-embed-text` through Ollama and stored in Chroma with its paper and page references. [Q15, T §2]

### FR-4 Paper cards

- **FR-4.1** A fixed pipeline (no model-chosen tool calls) turns the chunks into a paper card. The model never receives the whole paper collection in one prompt. [Q3, V §2]
- **FR-4.2** A card contains:
  - title, authors and year, each marked "unavailable" when missing or uncertain
  - findings relevant to the research question, each with references to supporting source passages
  - methods
  - limitations
  - 1–2 supporting quotes, each with a PDF page number [V §2]
- **FR-4.3** A paper's display name is its title, or the original upload filename when the title is "unavailable." [CQ3]
- **FR-4.4** Once saved, a card and its citations are never changed by later uploads. [V §2]

### FR-5 Quote verification

- **FR-5.1** Code checks that each quote appears on its cited PDF page, after joining words split by line-break hyphenation and collapsing whitespace. [V §2, T §6]
- **FR-5.2** If a quote fails, the model is asked once for a replacement quote, which is checked the same way. [Q5]
- **FR-5.3** A quote that still fails is kept on the card with a clear warning: "Not verified: not found on PDF page N of [display name]." It is never shown as verified. [Q5]
- **FR-5.4** If no quote on a card is verified, the card is still saved, with a visible card-level note: "No quotes could be verified." This is a warning, not a processing failure. [CQ2]
- **FR-5.5** Verification confirms only where a quote is located, not whether it is interpreted correctly. The interface does not claim otherwise. [V §2]
- **FR-5.6** Citations use 1-based PDF page numbers labeled "PDF page" (or "PDF p." in short tags). These may differ from page numbers printed in the paper. [V §2]
- **FR-5.7** A quote must be long enough to identify one passage: at least 8 words after normalization (default, confirmed in design). A shorter quote counts as failing verification. It goes through the same retry as FR-5.2, and if it still fails it shows the warning "Not verified: quote too short to locate reliably." This stops short, common phrases such as "the results" from being marked verified just because they appear on almost every page. [SC-2 review]

### FR-6 Comparison agent

- **FR-6.0** If the tracker has no sections yet (including when it holds only not-relevant papers), the comparison loop is skipped. One model call, with tools turned off, labels the paper either **New angle** (which starts the first section, with a broad section name per FR-8.2) or **Not relevant or relevance unclear**. It uses the same written relevance criteria as the comparison loop, so a paper's label does not depend on upload order. The reason cites passages from the new paper only. Invalid output is handled as a failed step (FR-11).
- **FR-6.1** After the card is written, a tool-calling loop compares the new paper with the saved collection. The loop is seeded with the most similar existing cards. Only papers that have a primary section are comparison targets. Not-relevant papers are never used as seeds. [Q3, stories review]
- **FR-6.2** Tools available to the model are read-only lookups of data already in the app: search passages, read a saved card, list sections. All three return only papers that have a primary section (and their passages), so the model never sees not-relevant papers. A label or citation that references a not-relevant paper is invalid output (FR-11). The model cannot touch files, run commands or use the internet. [T §5, Q3, stories review]
- **FR-6.3** The loop stops after a fixed cap of tool calls (between 6 and 8, set in design). [Q3]
- **FR-6.4** If the cap is reached without a valid classification, the app makes one more model call with tools turned off, asking for the classification from the evidence gathered so far. Invalid output from that call is handled as a failed step (FR-11). [CQ4]
- **FR-6.5** Passages cited in a classification must come from tool results or the seeded cards. Citations to passages the model did not retrieve are rejected as invalid output. [Q3]
- **FR-6.6** Existing cards are compared in small batches, not all at once. [V §2]
- **FR-6.7** Agreement or contradiction is assessed only when claims are comparable in methods, populations and conditions. Otherwise the relationship is "unclear." [V §2, V §7]

### FR-7 Classification

- **FR-7.1** Each paper gets exactly one primary label:
  - **New angle**: starts a new section for a subtopic, method or viewpoint not covered by existing sections.
  - **Adds to [section]**: adds evidence to an existing section, with a relationship of agrees, contradicts, extends or unclear. Independent evidence for an existing finding counts as an addition.
  - **Redundant with [paper]**: repeats an existing paper's relevant findings with no new population, method, condition or independent dataset. The paper is placed in the referenced paper's section and kept.
  - **Not relevant or relevance unclear**: no established connection to the research question. [V §2, Q2]
- **FR-7.2** Each relevant paper has one primary section. It may also list secondary "related sections," which are display-only links to sections that exist at upload time. [Q1, CQ1]
- **FR-7.3** Secondary sections are assigned only at upload. Earlier papers never gain links to sections created later. [CQ1]
- **FR-7.4** Every label has a one-sentence reason and references to the passages it relied on. A cross-paper label (adds to, redundant) cites evidence from both papers. Single-paper labels (new angle, not relevant) cite only the new paper, since there is no second paper to cite. [V §2]
- **FR-7.5** Labels describe overlap within the uploaded collection, not a paper's overall value or the state of the field. [V §2]
- **FR-7.6** Labels and sections cannot be edited by the user in the MVP. [V §3]

### FR-8 Tracker updates

- **FR-8.1** After a paper is classified, the app adds its card, creates or updates its section, and rewrites the short overview. [V §2]
- **FR-8.2** Section names are fixed once created. The prompt asks for broad section names so sections do not end up too narrow. [Q4]
- **FR-8.3** Section summaries and the overview may be rewritten, so they can reflect agreeing and contradicting findings. [Q4]
- **FR-8.4** Not-relevant papers are listed in a separate "Not relevant or relevance unclear" group with their reason. [V §2]
- **FR-8.5** Papers cannot be removed in the MVP. [Q9]

### FR-9 Coverage counts

- **FR-9.1** Paper count per section, using primary sections only. [V §2, Q1]
- **FR-9.2** Redundant papers count toward their assigned section. The number of redundant papers is also shown separately. [V §2]
- **FR-9.3** Not-relevant papers are left out of section counts but included in the total paper count. [V §2]
- **FR-9.4** Earliest and latest known publication years across the saved collection. Papers whose year is unavailable are counted and shown separately. [V §2]
- **FR-9.5** Counts are computed by code from saved data, never by the model. [R §2]

### FR-10 Background processing and progress

- **FR-10.1** The upload request returns immediately after validation, and processing continues in the background. [Q11]
- **FR-10.2** The page shows the current step: indexing text (chunking and embedding), writing card, comparing, updating tracker. It shows the result or the error when processing ends. [Q11]
- **FR-10.3** Only one paper is processed at a time (see FR-2.5). [CQ5]

### FR-11 Failure handling

- **FR-11.1** A failing background step (badly formatted model output, a model or embedding error, or an Ollama timeout) is retried up to 2 more times. Text extraction problems are not retried. Extraction happens during the upload request, and its problems are immediate rejections under FR-2.2, because the same file always gives the same text. [Q13, text-check timing decision]
- **FR-11.2** If the step still fails, nothing about the paper is saved. This covers the database rows, the Chroma embeddings, the stored PDF file and `tracker.md`. The tracker stays exactly as it was, and the page shows an error so the user can try again. [Q13]
- **FR-11.3** Because nothing is saved, including the file's hash, the user can upload the same PDF again after a failure without hitting duplicate detection. [CQ2, Q13]
- **FR-11.4** If the app stops while a paper is processing, any partial data for that paper is removed on the next start. [Q13, CQ5]
- **FR-11.5** Quote verification failures are warnings, not processing failures (see FR-5.4). [CQ2]
- **FR-11.6** When the start-up cleanup in FR-11.4 removes an interrupted paper, the tracker page shows a notice: "[original filename] was still processing when the app closed, so it was not saved. Upload it again to retry." The notice stays until the user dismisses it. The original filename is used because the title may not have been extracted yet. The temporary "processing" record that holds the filename is deleted once the notice is dismissed, so nothing about the paper remains. [Story plan clarification Q2]

### FR-12 Inspection view and citations

- **FR-12.1** The tracker page shows the overview, sections with their summaries and counts, each paper's card, its label and reason, secondary related sections, and the not-relevant group. [V §2, V §5]
- **FR-12.2** Citations on the page are short inline tags. Each shows the first author's surname and year when both are available, or the display name otherwise, plus "PDF p. N," for example "(Smith 2021, PDF p. 4)." Clicking a tag expands the cited passage. [Q6, CQ3]
- **FR-12.3** For each label and quote, the user can see the cited passage text, its PDF page, and a link that opens the stored PDF at that page in the browser's PDF viewer. [Q7]
- **FR-12.4** PDFs are served through a route that looks them up by paper ID, never by a file path supplied in the request. [Q7, T §5]
- **FR-12.5** Unverified quotes and the "No quotes could be verified" note are visibly different from verified quotes. [Q5, CQ2]

### FR-13 Markdown export

- **FR-13.1** `tracker.md` is rewritten from saved data, with no model call, after every successful upload. [Q10]
- **FR-13.2** It contains the question, the overview, the sections with summaries and counts, the cards, labels and reasons, the not-relevant group, the coverage counts, and the year range. [V §2]
- **FR-13.3** Citations in `tracker.md` are numbered footnotes ([1], [2], …). Each footnote gives the paper's display name, the PDF page and the quoted passage, and marks unverified quotes. [Q6]
- **FR-13.4** An "Export" action downloads the current `tracker.md`. [Q10]

### FR-14 Persistence

- **FR-14.1** Cards, sections, labels, reasons, citations, passages, embeddings and PDFs persist on local disk across restarts. Users never upload earlier papers again. [V §2]
- **FR-14.2** Structured data is stored in SQLite, passages and embeddings in Chroma, and PDFs and `tracker.md` in the data folder. [T §2]
- **FR-14.3** Saving and reopening the tracker, then adding a paper, never loses or alters earlier cards or sections (apart from the rewrites allowed by FR-8.3). [V §6]

### FR-15 Configuration and setup

- **FR-15.1** These settings are read from `.env`:
  - the Ollama model name
  - the context length (used by the app to size chunks and batches)
  - thinking mode on or off
  - the data folder (default `data/`) [Q14, Q16, CQ6]
- **FR-15.2** The repository includes an Ollama Modelfile that builds the default model from `qwen3:8b` with the required context size. Setup instructions add the matching `ollama create` command. [CQ8]
- **FR-15.3** Switching to a smaller fallback model (for example `qwen3:4b`) requires only changing `.env` and creating that model with a Modelfile. No code changes are needed. [Q14, CQ8]
- **FR-15.4** Setup is: clone the repository, `ollama pull qwen3:8b`, `ollama pull nomic-embed-text`, `ollama create` from the Modelfile, `uv sync`, then `uv run uvicorn research_tracker.main:app`. [T §2, Q15, CQ8]

### FR-16 Step timing

- **FR-16.1** The app logs how long each step takes (text extraction during the upload request, then indexing, writing card, comparing, updating tracker in the background) on every upload. [CQ7]
- **FR-16.2** Timing logs stay on the local disk. [T §2]

### FR-17 Evaluation script

- **FR-17.1** A command-line script (for example `uv run python -m research_tracker.evaluate`) runs the labeled test papers through the same processing pipeline as the web app. [Q17]
- **FR-17.2** The script uses its own `eval-data/` folder, cleared at the start of each run, and never touches the user's `data/` folder. `eval-data/` is listed in `.gitignore`. [CQ6]
- **FR-17.3** The script reports:
  - primary-label agreement with the team's manual labels
  - the stored-data quote check (FR-17.7)
  - whether section counts and the year range match the saved collection
  - a per-step timing summary from the logs [Q17, V §6, CQ7]
- **FR-17.4** Results can be opened in the web app by setting the data folder in `.env` to `eval-data/`. [CQ6]
- **FR-17.5** The format of the team's manual label file is decided in design.
- **FR-17.6** The script writes a quote checklist to `eval-data/`, listing every quote marked verified. Each entry has the paper's display name, the quote, its PDF page, and a link that opens the stored PDF at that page. A team member opens each link, confirms by eye whether the quote is on that page, and marks it found or not found. The script then reads the completed checklist and reports SC-2 from those marks. The person, not the app's own code, is what makes SC-2 an independent check. [SC-2 review]
- **FR-17.7** The script re-reads each quote, its PDF page and its verified flag from the saved database and from `tracker.md`, not from the pipeline's in-memory results, and checks them against the stored page text. It reports any quote whose shown flag, page or text differs from what is stored, or that fails the check. This catches wiring bugs, such as a replacement quote shown without being rechecked or the wrong page being displayed. [SC-2 review]

## 6. Non-Functional Requirements

### NFR-1 Security and privacy

The Security Baseline extension is off. The security rules from the technical document still apply as requirements. [Q18, T §5]

- **NFR-1.1** Uvicorn binds to `127.0.0.1` only. Ollama keeps its localhost default. Chroma telemetry is off (`anonymized_telemetry=False`).
- **NFR-1.2** No request leaves the laptop. The only model and embedding calls go to local Ollama.
- **NFR-1.3** Paper text is always given to the model as material to read, never as instructions. Tools are read-only lookups of uploaded content.
- **NFR-1.4** Jinja2 autoescaping stays on. Paper text and model output are never rendered as HTML.
- **NFR-1.5** All SQL uses `?` placeholders.
- **NFR-1.6** `.env`, `data/` and `eval-data/` are in `.gitignore`.
- **NFR-1.7** Upload checks follow FR-2.2 and FR-2.4.

### NFR-2 Performance

- **NFR-2.1** There is no fixed time target per paper for the MVP. Step times are measured (FR-16) on each teammate's laptop and reported. A target may be set later. [Q12]
- **NFR-2.2** Model calls are kept small through chunking and batched comparison, so they fit within the configured context length. [Q14, V §7]

### NFR-3 Reliability and data integrity

- **NFR-3.1** Processing a paper is all-or-nothing across SQLite, Chroma and the PDF folder (FR-11.2, FR-11.4).
- **NFR-3.2** Earlier cards and citations are never modified after they are saved (FR-4.4).
- **NFR-3.3** Structured data is not stored with pickle or shelve. [T §4]

### NFR-4 Testability

- **NFR-4.1** Tests use pytest and run with `uv run pytest`. They must pass before a pull request is merged. [T §3]
- **NFR-4.2** Unit tests use a fake model with pre-written answers, so results are fast and repeatable. Endpoint tests use FastAPI's `TestClient`. [T §3]
- **NFR-4.3** Model tests use the real Qwen3 and run only when asked for (for example with a pytest marker). [T §3]
- **NFR-4.4** Property-based tests (Partial mode: rules PBT-02, PBT-03, PBT-07, PBT-08 and PBT-09 are blocking) cover the pure functions: quote normalization and verification, chunking with page references, coverage counts, and any serialization round-trips. Property tests use Hypothesis as a dev dependency. [Q19]
- **NFR-4.5** Model behavior is checked through the fake-model unit tests and the evaluation script, not property tests. [Q19]
- **NFR-4.6** Quote verification has hand-written unit tests for hard cases, separate from the property tests and from the evaluation script. These include:
  - quotes below the minimum length
  - common short phrases
  - words with real hyphens (for example "self-report") next to words split by a line-break hyphen
  - quotes that run across a page break
  - the right quote on the wrong page
  - an empty quote, and page numbers out of range
  - a replacement quote, which is checked again after FR-5.2 [SC-2 review]

### NFR-5 Maintainability

- **NFR-5.1** No LangChain, LlamaIndex, hosted LLM SDKs, PyPDF2, PyMuPDF, OCR libraries, or pickle/shelve for tracker data. [T §4]
- **NFR-5.2** Prompts and the tool-call loop are plain Python functions the team writes, so they can be read and debugged. [T §4]
- **NFR-5.3** Python 3.12. Dependencies are declared in `pyproject.toml` and locked in `uv.lock`, both committed. [T §1]

### NFR-6 Usability

- **NFR-6.1** Error messages say what went wrong and what to do, for example "This PDF has too little extractable text; scanned PDFs are not supported."
- **NFR-6.2** Progress (FR-10.2) is visible during processing, which may take minutes.
- **NFR-6.3** The interface says that assessments cover only the uploaded papers. [V §1]

## 7. Out of Scope for the MVP

Imbalance notes and reading suggestions; manual editing of sections and labels; removing papers; scanned or image-only PDFs; non-English and non-PDF files; academic database search and paper recommendations; reference-manager integrations; accounts, cloud sync and shared trackers; multiple trackers in the interface; an upload queue; an in-app thinking-mode toggle; and generating a full literature review. [V §3, Q8, Q9, CQ5, Q16]

## 8. Success Criteria

These are measured on a manually labeled test set of up to 10 papers on one question, including redundant and contradicting pairs. [V §6]

| # | Criterion | Checked by |
|---|---|---|
| SC-1 | End-to-end demo: create a tracker, add papers, inspect evidence, reopen it, add another paper | Manual demo |
| SC-2 | 100% of quotes shown as verified are confirmed by a team member to appear on their cited PDF page | Manual checklist generated by the evaluation script (FR-17.6). Supported by the stored-data check (FR-17.7) and quote unit tests (NFR-4.6) |
| SC-3 | At least 80% of primary labels match the team's manual labels | Evaluation script |
| SC-4 | No earlier cards or sections lost after saving, reopening and adding a paper | Unit tests and the demo |
| SC-5 | Section counts and year range match the saved collection exactly | Unit tests and evaluation script |

## 9. Assumptions and Items for Later Stages

- **A-1** The language is not detected automatically. Users are told the app supports English papers only. [V §2]
- **A-2** The exact tool-call cap (6–8), chunk size, batch size, the "too little text" threshold, the minimum quote length (default 8 words) and the context size in the Modelfile are set during design and tuned with the evaluation script.
- **A-3** Thinking mode is applied through whichever mechanism Ollama's OpenAI-compatible API supports for Qwen3 (an API option or Qwen3's `/think` and `/no_think` prompt switch). This is confirmed during design. [Q16]
- **A-4** The context length in `.env` must match the context size built into the Modelfile. The setup instructions say so, and a fallback model needs its own Modelfile. [CQ8]
- **A-5** Opening a PDF at a cited page relies on the browser's built-in PDF viewer. Some browsers (notably Safari) may not jump to the page. The cited passage is always shown next to the link, so evidence stays readable. [Q7]
- **A-6** The publication-year range covers all saved papers, including not-relevant ones, following the vision's wording "across the saved collection." [V §2]

## 10. Extension Configuration

| Extension | Status | Effect |
|---|---|---|
| Security Baseline | Off | The technical document's security rules (NFR-1) still apply |
| Property-Based Testing | Partial | PBT-02, PBT-03, PBT-07, PBT-08 and PBT-09 are blocking; other PBT rules are advisory |
| Resiliency Baseline | Off | Failure handling is covered by FR-11 and NFR-3 |
