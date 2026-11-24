# Components: Research Tracker

**Decisions applied**: application-design-plan.md Q1–Q11 and follow-up 1. Business rules (exact validation logic, prompt wording, thresholds, table columns) are left to each unit's Functional Design.

## Package Layout

The code is organized by component (Q1: A). Each component has one module or subpackage, so owners mostly edit their own files.

```text
research_tracker/
    main.py              # app factory and start-up sequence (Web + Job Runner wiring)
    settings.py          # 1  Settings
    models.py            #    Shared data types (see below)
    pdf_intake.py        # 2  PDF Intake
    indexer.py           # 3  Indexer
    model_client/        # 4  Model Client
        base.py          #    ModelClient interface
        ollama.py        #    real client (openai library -> Ollama)
        fake.py          #    scripted fake for tests
    cards.py             # 5  Card Writer
    quotes.py            # 6  Quote Checker (pure functions + replacement request)
    agent/               # 7  Comparison Agent, 8 Agent Tools
        comparison.py
        tools.py
        validation.py
        prompts.py
    tracker_updater.py   # 9  Tracker Updater
    coverage.py          # 10 Coverage (pure functions)
    export.py            # 11 Markdown Exporter
    storage/             # 12 Storage
        db.py            #    SQLite
        schema.sql
        passages.py      #    Chroma
        files.py         #    PDFs, tracker.md, timing log
        store.py         #    TrackerStore facade: reads, commit, cleanup
    timing.py            #    step timer used by the Processor
    processor.py         # 13 Paper Processor (the only orchestrator)
    jobs.py              # 14 Job Runner
    web/                 # 15 Web
        routes.py
        templates/       #    Jinja2 (autoescape on)
        static/poll.js   #    the only JavaScript (progress polling)
    evaluate/            # 16 Evaluation
        __main__.py      #    uv run python -m research_tracker.evaluate
        report.py
        checklist.py
        stored_check.py
Modelfile                # repository root (FR-15.2)
.env.example
```

## Shared Data Types (`models.py`)

These types cross component boundaries. They are Pydantic models (Q6), so the same definitions validate model output, move data between components, and serialize to SQLite.

| Type | Purpose | Key fields (types refined in Functional Design) |
|---|---|---|
| `ExtractedPdf` | Result of a successful upload check | `sha256`, `original_filename`, `data: bytes`, `pages: list[str]` (index 0 = PDF page 1) |
| `Chunk` | A piece of text with page references | `chunk_id`, `pdf_pages: list[int]`, `text` |
| `PassageRef` | A pointer to a source passage | `paper_id` (or `NEW` for the paper being processed), `chunk_id`, `pdf_page` |
| `Passage` | A passage returned to the model or shown to the user | `PassageRef` fields + `display_name`, `text` |
| `Quote` | A card quote and its check result | `text`, `pdf_page`, `status` (`verified`, `not_found`, `too_short`), `replaced: bool` |
| `Finding` | A finding relevant to the question | `text`, `passages: list[PassageRef]` |
| `Card` | A paper card | `title`, `authors`, `year` (each value or "unavailable"), `findings`, `methods`, `limitations`, `quotes: list[Quote]` |
| `LabelKind` | Enum | `new_angle`, `adds_to`, `redundant`, `not_relevant` |
| `Relationship` | Enum (for `adds_to`) | `agrees`, `contradicts`, `extends`, `unclear` |
| `Classification` | The validated label | `kind`, `section_id` or `new_section_name`, `redundant_with_paper_id`, `relationship`, `related_section_ids`, `reason`, `citations: list[PassageRef]` |
| `Section` | A tracker section | `section_id`, `name` (fixed), `summary` |
| `TrackerUpdate` | Section and overview changes to save | `new_section` or `updated_section_id`, `section_summary`, `overview` |
| `PaperDraft` | Everything about one paper, held in memory until commit (Q4) | `extracted`, `chunks`, `chunk_embeddings`, `card_embedding`, `card`, `classification`, `tracker_update` |
| `PaperRecord` | A committed paper as read back | `paper_id`, `display_name`, `original_filename`, `card`, `classification`, `primary_section_id` |
| `TrackerView` | Everything the tracker page and export need | `tracker`, `sections`, `papers`, `not_relevant`, `coverage`, `notices` |
| `Coverage` | Counts and year range | `section_counts`, `redundant_count`, `total_count`, `earliest_year`, `latest_year`, `unknown_year_count` |
| `JobStatus` | Progress shown on the page | `state` (`idle`, `processing`, `done`, `failed`), `step`, `original_filename`, `error`, `paper_id` |
| `Notice` | Interrupted-paper notice (FR-11.6) | `notice_id`, `original_filename` |

---

## 1. Settings (`settings.py`)

- **Purpose**: one typed place for every setting.
- **Responsibilities**:
  - read `.env` with `pydantic-settings` (Q10): model name, context length, thinking mode, data folder, Ollama URL
  - hold the tuning values set in Functional Design (A-2): chunk size, batch size, model-call cap, minimum quote length, "too little text" threshold, seed count
  - fail at start-up with a message naming any missing or invalid setting (US-02)
  - reject an Ollama URL that is not on localhost (C-2)
- **Interface**: `Settings` object passed to the components that need it. Nothing reads environment variables directly.
- **Stories**: US-02, US-29

## 2. PDF Intake (`pdf_intake.py`)

- **Purpose**: decide, during the upload request, whether a file can become a paper (FR-2.2).
- **Responsibilities**:
  - check the `%PDF-` marker, size, encryption, page count, the 10-paper limit and extractable text
  - extract text page by page with `pypdf`
  - compute SHA-256 and check for duplicates against committed papers
  - raise a `PdfRejected` error with a plain-language message
- **Interface**: `check_upload(...) -> ExtractedPdf` or `PdfRejected`.
- **Stories**: US-04, US-05, US-06

## 3. Indexer (`indexer.py`)

- **Purpose**: turn pages into chunks with page references, and embed them.
- **Responsibilities**:
  - chunk pages within the size set in Settings
  - embed chunks and the card text through the Model Client
  - search the new paper's own chunks in memory, for the `search_passages` tool with `scope="new"`
- **Interface**: `chunk_pages`, `embed_chunks`, `embed_text`, `InMemoryPassages`.
- **Stories**: US-04

## 4. Model Client (`model_client/`)

- **Purpose**: the only route to Ollama (C-2), and the seam where the fake model replaces it (Q5).
- **Responsibilities**:
  - **chat**: messages, optional tools, optional output schema, and thinking on or off
  - **embeddings**
  - turn transport failures into one `ModelError` type
  - the real client uses the `openai` library pointed at local Ollama
  - **the fake client** returns scripted answers per step name, taken from a list in order (Q5 addition). It raises an error when a step's list runs out, and returns fixed embeddings
- **Interface**: `ModelClient` protocol with `chat(...)` and `embed(...)`. Every component receives a `ModelClient`; none imports `openai`.
- **Stories**: all model-dependent stories; NFR-4.2

## 5. Card Writer (`cards.py`)

- **Purpose**: produce a validated `Card` with a fixed sequence of model calls (FR-4.1).
- **Responsibilities**:
  - read the chunks in batches that fit the context, and collect relevant findings with passage references
  - assemble the card and validate it against its Pydantic schema
  - run every quote through the Quote Checker, including the one replacement request
- **Interface**: `write_card(...) -> Card`.
- **Stories**: US-09, US-10

## 6. Quote Checker (`quotes.py`)

- **Purpose**: decide, in code, whether a quote is on its cited page (FR-5, C-7).
- **Responsibilities**:
  - normalize text (line-break hyphens, whitespace)
  - check the minimum length
  - verify the quote against the page text
  - ask the model once for a replacement and check it the same way
  - shared by the Card Writer and the Evaluation stored-data check, so both use one definition
- **Interface**: pure `normalize`, `check_quote`; plus `check_with_replacement(...)` (calls the model).
- **Stories**: US-10, US-26, US-27; NFR-4.4, NFR-4.6

## 7. Comparison Agent (`agent/comparison.py`, `validation.py`, `prompts.py`)

- **Purpose**: give each new paper exactly one valid `Classification`.
- **Responsibilities**:
  - **No sections yet**: make the single tool-free call (FR-6.0)
  - **Otherwise**, run the loop:
    - seed it with similar cards of committed papers that have a section
    - use native tool calling (Q7)
    - count model calls toward the cap, re-asking a failed turn up to 2 times (FR-6.3)
    - make the final tool-free call, retrying only that call (FR-6.4)
  - validate the result: the label kind, that the section or paper it names exists and has a section, that every citation was retrieved, and the both-papers rule for cross-paper labels
  - keep paper text out of the instructions (C-3): paper text goes only in clearly marked content, never in the system message
- **Interface**: `classify(...) -> Classification` or `ComparisonFailed`.
- **Stories**: US-11, US-12, US-13, US-14, US-15

## 8. Agent Tools (`agent/tools.py`)

- **Purpose**: the read-only lookups the model may call (FR-6.2).
- **Responsibilities**:
  - provide `search_passages`, `read_card` and `list_sections`
  - return only committed papers that have a primary section (Q4, FR-6.2)
  - `search_passages` can also search the new paper's own chunks in memory
  - record every passage returned, so the Agent can check citations (FR-6.5)
- **Interface**: `ToolBox` with the three tools, their JSON schemas for the model, and a `seen_passages` set.
- **Stories**: US-12, US-14

## 9. Tracker Updater (`tracker_updater.py`)

- **Purpose**: work out the section and overview changes for one classified paper (FR-8).
- **Responsibilities**:
  - choose the new or existing section
  - rewrite the section summary in small batches
  - rewrite the overview
  - never rename a section
  - return the changes, without saving them
- **Interface**: `plan_update(...) -> TrackerUpdate`.
- **Stories**: US-19

## 10. Coverage (`coverage.py`)

- **Purpose**: counts and year range, computed by code from saved data (FR-9, C-7).
- **Responsibilities**: section counts (primary section only), redundant count, total count, year range, unknown-year count.
- **Interface**: pure `compute_coverage(papers) -> Coverage`.
- **Stories**: US-20, US-21, US-25; NFR-4.4

## 11. Markdown Exporter (`export.py`)

- **Purpose**: render `tracker.md` from saved data (FR-13), with no model call (C-7).
- **Responsibilities**:
  - render the question, overview, sections, cards, labels, the not-relevant group, coverage and the year range
  - number the footnotes and mark unverified quotes
- **Interface**: pure `render_markdown(view) -> str`. The file is written through Storage.
- **Stories**: US-22, US-27

## 12. Storage (`storage/`)

- **Purpose**: everything kept on disk, plus the all-or-nothing save (Q4: A).
- **Responsibilities**:
  - **`db.py`**:
    - SQLite with one connection per thread (Q2)
    - `?` placeholders only (C-5)
    - schema from `schema.sql` with a schema version number; refuses to start on a version mismatch, with a clear message (Q11 addition)
  - **`passages.py`**:
    - Chroma, with telemetry off (C-1)
    - one collection for passages and one for card embeddings, each entry tagged with `paper_id`
    - every query is filtered to a set of allowed paper IDs
    - Chroma never embeds anything itself, so it never downloads its default model (C-2):
      - both collections are created with an explicit embedding function that raises an error if called
      - entries are always added with precomputed `embeddings=` and searched with `query_embeddings=`
      - raw text is stored only as data, never passed for Chroma to embed (`query_texts=` is never used)
      - a test checks that both collections use the refusing function and that adding or querying without vectors raises the error
      - the exact behavior is confirmed against the installed Chroma version in Functional Design
  - **`files.py`**: PDFs named by SHA-256, `tracker.md`, and the timing log, all under the data folder
  - **`store.py`** (`TrackerStore`):
    - read methods that return committed data only
    - `commit(draft)`: PDF file and Chroma entries first, then one SQLite transaction that saves every row and marks the paper committed
    - start-up cleanup of anything not committed
    - the processing-job record and notices (FR-11.6), which are saved outside the paper's commit
- **Interface**: `TrackerStore` (see component-methods.md).
- **Stories**: US-08, US-23; NFR-3.1

## 13. Paper Processor (`processor.py`), a service

- **Purpose**: run one paper from `ExtractedPdf` to a committed paper. It is the only component that calls the others in sequence (Q1).
- **Responsibilities**:
  - run the steps in order: indexing, writing the card, comparing, updating the tracker, then committing and writing `tracker.md`
  - apply step retries (FR-11.1)
  - report progress through a callback
  - time each step (FR-16)
  - on failure, discard the in-memory draft and make sure nothing is left on disk
- **Interface**: `process(...) -> PaperRecord` or `ProcessingFailed`. Details in services.md.
- **Stories**: US-04, US-08; used by US-24

## 14. Job Runner (`jobs.py`)

- **Purpose**: run one paper at a time in the background (Q2: B) and own "is a paper processing?" (FR-2.5).
- **Responsibilities**:
  - start one worker thread with the app
  - `submit` saves the processing-job record and starts the work, or raises `Busy`
  - expose the current `JobStatus` for polling (Q3)
  - at start-up, run the cleanup and turn interrupted jobs into notices
- **Interface**: `JobRunner` (see component-methods.md).
- **Stories**: US-04, US-07, US-08

## 15. Web (`web/`, `main.py`)

- **Purpose**: the local pages and endpoints.
- **Responsibilities**:
  - FastAPI routes: create the tracker, the tracker page, upload, status, PDF by paper ID, export, dismiss notice
  - Jinja2 templates with autoescape on (C-4), and `<details>` for citations (Q8)
  - `poll.js` for progress (Q3)
  - map errors to plain messages (C-6)
  - **start-up** in `main.py`, in this order:
    1. load Settings
    2. check the schema
    3. run cleanup
    4. regenerate `tracker.md`
    5. start the Job Runner
  - Uvicorn binds to `127.0.0.1` (C-1)
- **Interface**: HTTP routes (see component-methods.md).
- **Stories**: US-01, US-03, US-05 to US-08, US-16 to US-23

## 16. Evaluation (`evaluate/`)

- **Purpose**: measure the app against SC-2, SC-3 and SC-5, and report timing (FR-17).
- **Responsibilities**:
  - clear `eval-data/` and build the same Processor with settings pointing there (Q9: A)
  - read the label file and process the papers in upload order, skipping the Job Runner
  - write the report and the quote checklist
  - score a completed checklist
  - run the stored-data quote check
  - summarize timing and record the settings used
- **Interface**: command line `uv run python -m research_tracker.evaluate run|score-checklist`.
- **Stories**: US-24 to US-29

---

## Story-to-Component Check

Every story has at least one owning component.

| Story | Components |
|---|---|
| US-01 | Web (`main.py`), Settings, README/Modelfile |
| US-02 | Settings |
| US-03 | Web, Storage |
| US-04 | Web, PDF Intake, Job Runner, Processor, Indexer |
| US-05 | PDF Intake, Web |
| US-06 | PDF Intake, Storage |
| US-07 | Job Runner, Web |
| US-08 | Processor, Job Runner, Storage, Web |
| US-09 | Card Writer |
| US-10 | Quote Checker, Card Writer, Web |
| US-11 | Comparison Agent |
| US-12 | Comparison Agent, Agent Tools |
| US-13 | Comparison Agent, Tracker Updater |
| US-14 | Comparison Agent, Agent Tools, Web |
| US-15 | Comparison Agent, Web |
| US-16 | Web |
| US-17 | Web, Storage |
| US-18 | Web |
| US-19 | Tracker Updater, Web |
| US-20 | Coverage, Web |
| US-21 | Coverage, Web |
| US-22 | Markdown Exporter, Storage, Web |
| US-23 | Storage, Web |
| US-24 | Evaluation, Processor |
| US-25 | Evaluation, Coverage |
| US-26 | Evaluation |
| US-27 | Evaluation, Quote Checker, Markdown Exporter |
| US-28 | Processor (timing), Evaluation |
| US-29 | Settings, Evaluation |
