# Application Design Plan: Research Tracker

This stage defines the app's components, their methods and how they connect. Detailed business rules come later, in each unit's Functional Design. The answers here also become the "shared data formats and tool interfaces" the README asks the team to agree on before implementation.

Answer the questions in Part B. Once the answers are complete and checked, I'll carry out Part C.

---

## Part A: Proposed Components (for reference while answering)

These follow the requirements and stories. The questions below may change them.

| # | Component | Main job | Main stories |
|---|---|---|---|
| 1 | Settings | Read `.env` (model, context length, thinking mode, data folder) and the tuning values | US-02, US-29 |
| 2 | PDF Intake | Upload checks, SHA-256, `pypdf` page-by-page extraction, rejections | US-04, US-05, US-06 |
| 3 | Indexer | Chunk pages with page references; embed chunks | US-04 |
| 4 | Model Client | The only path to Ollama: chat, tool calls, embeddings, plus the fake model for tests | all model stories |
| 5 | Card Writer | Fixed pipeline that turns chunks into a card | US-09 |
| 6 | Quote Checker | Normalize, check minimum length, verify on page, one replacement request | US-10, US-26, US-27 |
| 7 | Comparison Agent | First-paper call (FR-6.0), capped tool loop, final tool-free call, output validation | US-11 to US-15 |
| 8 | Agent Tools | Read-only lookups over relevant papers only: search passages, read card, list sections | US-12, US-14 |
| 9 | Tracker Updater | Create or update the section, rewrite section summary and overview | US-19 |
| 10 | Coverage | Section, redundant and total counts; year range (pure functions) | US-20, US-21 |
| 11 | Markdown Exporter | Render `tracker.md` from saved data (no model) | US-22 |
| 12 | Storage | SQLite (structured data), Chroma (passages and embeddings), files (PDFs, `tracker.md`, timing logs) | US-23 |
| 13 | Paper Processor (service) | Runs one paper through every step with retries; all-or-nothing save; progress; timing | US-04, US-08 |
| 14 | Job Runner | Runs one paper at a time in the background; blocks new uploads; start-up cleanup and notice | US-07, US-08 |
| 15 | Web | FastAPI routes and Jinja2 pages: create tracker, tracker page, upload, status, PDF by ID, export, dismiss notice | US-03, US-16 to US-19 |
| 16 | Evaluation | Run the test set into `eval-data/`, report, quote checklist, stored-data check, timing summary | US-24 to US-29 |

---

## Part B: Design Questions

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe what you want.

## Question 1: Code organization

How should the `research_tracker` package be organized?

A) By component: one module or subpackage per component in Part A (for example `pdf_intake.py`, `quotes.py`, `agent/`, `storage/`, `web/`), with the Paper Processor as the only place that calls several components in sequence

B) By layer: `routes/`, `services/`, `domain/`, `storage/`, with each component split across layers

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 2: Background processing

How should the background work (FR-10) run?

A) FastAPI's built-in `BackgroundTasks`: the upload endpoint schedules the Paper Processor after sending its response. Simplest, with no extra code to manage

B) One dedicated worker thread started with the app, which takes one paper at a time. Gives the app one clear place for "is a paper processing?" and makes shutdown handling explicit

C) An `asyncio` task with async model calls. Needs the async `openai` client and async-safe storage code

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 3: Progress on the page

How should the tracker page learn about progress and the result (FR-10.2)?

A) The page asks a small status endpoint every few seconds (polling) with a little plain JavaScript, and reloads when processing ends

B) Server-sent events: the server pushes each step change to the page as it happens

C) No JavaScript: the page reloads itself every few seconds with an HTML refresh tag while a paper is processing

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 4: Saving all-or-nothing across three stores

A paper touches SQLite, Chroma and the PDF folder. If anything fails, nothing may remain (FR-11.2), including after a crash (FR-11.4). How should saving work?

A) **Do the work in memory, commit at the end, with SQLite as the single source of truth.** All model steps run on in-memory data. At the end, the PDF file and the Chroma entries (tagged with the paper ID) are written first. Then one SQLite transaction saves every row and marks the paper as committed. Everything the app reads is filtered to committed papers. Start-up cleanup deletes any file or Chroma entry whose paper is not committed in SQLite

B) **Write as you go with a "pending" status.** Each step saves its results right away, marked pending. Success flips the paper to committed. Failure or start-up cleanup deletes everything still pending. Easier to inspect half-done work, but every query must remember to skip pending data

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 5: Model client and fake model

How should code talk to the model, so that tests can swap in the fake model (NFR-4.2)?

A) One `ModelClient` interface with `chat(...)` and `embed(...)`. The real version calls Ollama through the `openai` library. The fake version returns scripted answers keyed by step name (for example "card", "quote_retry", "first_label", "compare_turn", "final_label", "section_summary", "overview"), and returns fixed embeddings. Components receive the client as a parameter; none of them import `openai` directly

B) Same interface, but the fake picks its answer by matching text in the prompt

C) Patch the `openai` library in tests instead of using an interface

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 6: Structured model output

How should the app get cards and labels from the model in a reliable format?

A) Define each output as a Pydantic model (Pydantic already comes with FastAPI). Ask Ollama to follow the matching JSON schema where its OpenAI-compatible API supports it, then always validate with Pydantic. Output that fails validation is a failed step (FR-11). Whether Ollama enforces the schema is confirmed in Functional Design

B) Ask for JSON in the prompt only, and validate with Pydantic

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 7: Tool calling in the comparison loop

How should the model call the read-only tools (FR-6.2)?

A) Native tool calling: pass the tools through the `openai` library's `tools` parameter, which Ollama supports for Qwen3. Our own loop runs each tool call and counts calls against the cap

B) Our own text protocol: the model replies with a JSON "action" (tool name and arguments) that our code parses. Works with any model, but needs more prompt and parsing code

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 8: Web page technology

How should the pages be built (FR-12)?

A) Server-rendered Jinja2 pages with plain HTML and CSS. Use the HTML `<details>` element so citations expand without JavaScript, and add JavaScript only for progress polling, if Question 3 needs it. No front-end framework or build step

B) Jinja2 pages plus htmx (one small JavaScript file kept in the repository) for partial page updates

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 9: How the evaluation runs the pipeline

How should the evaluation script (FR-17.1) run papers "through the same pipeline as the web app"?

A) In the same Python process: it builds the same Paper Processor with settings pointing at `eval-data/`, and calls it directly for each paper in upload order. No server needed

B) Over HTTP: it starts the web server and uploads each paper through the real endpoints, waiting for each to finish

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 10: Loading settings

How should `.env` settings be read (FR-15.1)?

A) `pydantic-settings`: a typed settings class that reads `.env`, checks types at start-up and names any missing or invalid setting (US-02). Adds one small dependency

B) `python-dotenv` plus our own checks in a plain dataclass

C) Read environment variables with `os.environ` only, with no `.env` loader

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 11: Database schema setup

How should the SQLite tables be created and changed over time?

A) A single `schema.sql` run at start-up with `CREATE TABLE IF NOT EXISTS`, plus a schema version number. A schema change during the MVP means deleting `data/` and starting over, which is acceptable before the deadline

B) Numbered migration scripts that upgrade an existing database in place, so testers keep their data across schema changes

X) Other (please describe after [Answer]: tag below)

[Answer]:

---

## Part C: Execution Checklist

This runs after the answers are complete and checked.

- [ ] **1. Prepare**
  - [ ] 1.1 Apply the Part B answers to the component list in Part A
  - [ ] 1.2 Re-check every story (US-01 to US-29) against the components, so each story has an owning component
- [ ] **2. `components.md`**
  - [ ] 2.1 For each component: purpose, responsibilities, interfaces, stories served
  - [ ] 2.2 State the shared data types that cross components (for example Page, Chunk, Card, Quote, Label, Section, PaperRecord, ProgressState)
- [ ] **3. `component-methods.md`**
  - [ ] 3.1 Method signatures with input and output types for each component
  - [ ] 3.2 A one-line purpose for each method; business rules left to Functional Design
  - [ ] 3.3 The tool interface the model sees (names, arguments, results)
- [ ] **4. `services.md`**
  - [ ] 4.1 The Paper Processor flow step by step: steps, retries, progress, timing, commit
  - [ ] 4.2 The Job Runner: one-at-a-time rule, start-up cleanup, interrupted-paper notice
  - [ ] 4.3 The Evaluation flow
- [ ] **5. `component-dependency.md`**
  - [ ] 5.1 Dependency matrix (who calls whom), with no loops
  - [ ] 5.2 Data flow for an upload, from request to `tracker.md`, as a diagram with a text alternative
  - [ ] 5.3 Where each shared constraint (C-1 to C-7) is enforced
- [ ] **6. `application-design.md`**
  - [ ] 6.1 Combine the four documents into one overview
- [ ] **7. Check**
  - [ ] 7.1 Every story and every FR has an owning component
  - [ ] 7.2 Interfaces are consistent across documents
  - [ ] 7.3 Note the pure functions that need property-based tests (PBT Partial)
  - [ ] 7.4 Update `aidlc-state.md` and `audit.md`, and present for approval
