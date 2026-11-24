# User Stories: Research Tracker (MVP)

**Sources**: `requirements.md` (FR, NFR, SC and A IDs), `personas.md` (P1, P2), `story-generation-plan.md` (format decisions)

## How to Read These Stories

- **Personas**: P1 Student Researcher, P2 Team Evaluator (see `personas.md`).
- **Epics** follow the user journey: E1 Set up and start → E2 Create a tracker → E3 Upload a paper → E4 Read the card and label → E5 Inspect evidence → E6 See the tracker and its coverage → E7 Export and return later → E8 Evaluate the app.
- **Acceptance criteria**: Given / When / Then for behavior that involves the model or several steps; a checklist for simple rules.
- **Model output**: criteria check structure and rules only, never a specific label or wording. Whether labels are *correct* is measured by the evaluation script (SC-3).
- **Demo path**: the story is needed for the SC-1 end-to-end demo (create a tracker, add papers, inspect evidence, reopen, add another paper).
- **Depends on**: hard dependencies only, meaning another story whose behavior must exist before this one can be seen working. Every story can still be tested on its own with seeded data, the fake model and fake embeddings, and FastAPI's `TestClient`.
- **Covers**: the requirement IDs the story is accountable for. See the coverage table at the end.

---

## Shared Constraints (apply to every story)

Every story's acceptance criteria implicitly include these. They are listed once here instead of in each story (story plan Q9).

| ID | Constraint | Source |
|---|---|---|
| C-1 | The app listens only on `127.0.0.1`. Ollama stays on its localhost default. Chroma telemetry is off | NFR-1.1 |
| C-2 | No request leaves the laptop. The only model and embedding calls go to local Ollama | NFR-1.2 |
| C-3 | Paper text is always given to the model as material to read, never as instructions. Model tools are read-only lookups of uploaded content and cannot touch files, run commands or use the internet | NFR-1.3, FR-6.2 |
| C-4 | Jinja2 autoescaping stays on. Paper text and model output are shown as plain text, never run as HTML | NFR-1.4 |
| C-5 | All SQL uses `?` placeholders | NFR-1.5 |
| C-6 | Error messages say what went wrong and what to do next, in plain language | NFR-6.1 |
| C-7 | Counts, quote checks and the Markdown export are done by code, never by the model | FR-9.5, FR-5.1, FR-13.1 |

---

## E1: Set up and start

### US-01: Install and start the app

**Persona**: P1 · **Demo path** · **Depends on**: none
**Covers**: FR-15.2, FR-15.4, NFR-5.3

As a student researcher, I want to install and start the app by following the README, so that I can use it on my own laptop without accounts or hosting.

**Acceptance criteria**
- [ ] The README lists the setup commands in order: clone, `ollama pull qwen3:8b`, `ollama pull nomic-embed-text`, `ollama create` from the Modelfile in the repository, `uv sync`, `uv run uvicorn research_tracker.main:app`
- [ ] The repository includes the Modelfile, which builds the default model from `qwen3:8b` with the required context size
- [ ] After setup, opening `http://127.0.0.1:8000` shows the app
- [ ] The README says the app supports English, text-based PDFs only, up to 10 papers and 20 pages each (A-1)
- **Given** Ollama is not running or the model has not been created, **when** the student uploads a paper, **then** the page shows an error that names the missing piece (for example "Cannot reach Ollama at localhost:11434. Start Ollama and try again"), and nothing is saved

### US-02: Switch to a smaller model on a slower laptop

**Persona**: P1 · **Depends on**: US-01
**Covers**: FR-15.1, FR-15.3, A-4

As a student researcher with a slower laptop, I want to switch to a smaller local model by changing settings, so that the app still runs well enough without code changes.

**Acceptance criteria**
- [ ] `.env` holds the model name, the context length, thinking mode on or off, and the data folder (default `data/`)
- [ ] Changing the model name in `.env` and restarting the app makes all model calls use the new model
- [ ] The README explains how to create a fallback model (for example from `qwen3:4b`) with its own Modelfile
- [ ] The README says the context length in `.env` must match the context size in the Modelfile
- [ ] Changing settings does not require any code change
- [ ] A missing or invalid setting stops the app at start-up with a message naming the setting

---

## E2: Create a tracker

### US-03: Create a tracker from my research question

**Persona**: P1 · **Demo path** · **Depends on**: US-01
**Covers**: FR-1.1, FR-1.2, FR-1.4

As a student researcher, I want to create a tracker by entering my research question, so that every paper I add is judged against that question.

**Acceptance criteria**
- [ ] With no tracker saved, opening the app shows a "create tracker" page asking for a research question
- [ ] Submitting a question saves a tracker with that question, an empty overview and no sections, then opens the tracker page
- [ ] An empty question is rejected with a message asking for one
- [ ] Once a tracker exists, the "create tracker" page is no longer offered (one tracker only)
- [ ] Tracker data and API paths include a tracker ID (for example `/api/trackers/{tracker_id}/papers`)

---

## E3: Upload a paper

### US-04: Upload a paper and watch it being processed

**Persona**: P1 · **Demo path** · **Depends on**: US-03
**Covers**: FR-2.1, FR-2.4, FR-3.1, FR-3.2, FR-3.3, FR-3.4, FR-10.1, FR-10.2, FR-16.1, NFR-6.2

As a student researcher, I want to upload one PDF and see which step it is on, so that I know the app is working while the model takes several minutes.

**Acceptance criteria**
- **Given** a tracker with fewer than 10 papers and no paper processing, **when** the student uploads a valid PDF, **then** all upload checks and text extraction finish during the request, the request returns without waiting for the model, and the page starts showing progress
- **Given** a paper is processing, **when** the student watches the page, **then** it shows the current step in order: indexing text, writing card, comparing, updating tracker
- **Given** processing finishes, **when** the page updates, **then** it shows the new card and label without a manual reload
- [ ] Only one PDF can be chosen per upload
- [ ] Text is extracted page by page with `pypdf`, keeping 1-based PDF page numbers
- [ ] Each chunk records its PDF page or pages, and each embedding in Chroma is linked to its paper and pages
- [ ] The stored PDF file is named by its SHA-256 hash. The original filename is saved for display only
- [ ] The time of each step (extraction in the request, then the four background steps) is written to a local log

### US-05: Get a clear reason when my upload is rejected

**Persona**: P1 · **Depends on**: US-03
**Covers**: FR-2.2, NFR-1.7, NFR-6.1, A-1

As a student researcher, I want to be told right away why a file was not accepted, so that I can fix it or choose another paper.

**Acceptance criteria**
- [ ] Each case below is rejected immediately on the upload form, before any background work, with nothing saved:
  - [ ] a file that does not start with `%PDF-` ("Upload a PDF file")
  - [ ] a file over 25 MB
  - [ ] an encrypted PDF
  - [ ] a PDF over 20 pages
  - [ ] a PDF with too little extractable text, for example "This PDF has too little extractable text; scanned PDFs are not supported."
  - [ ] an upload when the tracker already holds 10 papers
- [ ] Each message says what is wrong and what limit applies
- [ ] After a rejection, the tracker is exactly as it was and the student can upload another file straight away

### US-06: Be stopped from adding the same PDF twice

**Persona**: P1 · **Depends on**: US-04
**Covers**: FR-2.3

As a student researcher, I want the app to tell me when I upload a PDF I already added, so that the same paper is not counted twice.

**Acceptance criteria**
- [ ] An upload whose SHA-256 hash matches a saved paper is rejected with "This PDF is already in the tracker"
- [ ] The check runs before any model call
- [ ] Nothing is saved and the tracker is unchanged
- [ ] A different file of the same paper (for example another version) is not detected. This is expected in the MVP

### US-07: Be told to wait while a paper is still processing

**Persona**: P1 · **Depends on**: US-04
**Covers**: FR-2.5, FR-10.3

As a student researcher, I want to be told to wait if I try to upload while another paper is processing, so that papers are handled one at a time in a predictable order.

**Acceptance criteria**
- [ ] While a paper is processing, the upload control on the page is disabled
- [ ] An upload request sent anyway (for example from an old browser tab or directly to the API) is rejected by the server with a message such as "Wait until the current paper finishes"
- [ ] The paper already processing is not affected
- [ ] Once processing ends, successfully or not, uploading is available again

### US-08: Try again when processing fails

**Persona**: P1 · **Depends on**: US-04
**Covers**: FR-11.1, FR-11.2, FR-11.3, FR-11.4, FR-11.6, NFR-3.1

As a student researcher, I want a failed or interrupted paper to leave my tracker untouched and tell me to try again, so that I never end up with a half-saved paper I cannot remove.

**Acceptance criteria**
- **Given** a background step fails (badly formatted model output, a model or embedding error, or an Ollama timeout), **when** it fails, **then** the step is retried up to 2 more times before giving up
- **Given** a step still fails after its retries, **when** processing stops, **then** nothing about the paper remains in SQLite, Chroma, the PDF folder or `tracker.md`, the tracker looks exactly as before, and the page shows an error saying the paper was not added and can be uploaded again
- **Given** processing failed for a PDF, **when** the student uploads the same PDF again, **then** it is not rejected as a duplicate
- **Given** the app was closed while a paper was processing, **when** the app starts again, **then** all partial data for that paper is removed, and the tracker page shows "[original filename] was still processing when the app closed, so it was not saved. Upload it again to retry."
- **Given** that notice is showing, **when** the student dismisses it, **then** the notice disappears and the temporary processing record is deleted, so nothing about the paper remains
- [ ] A quote that fails verification does not count as a processing failure (see US-10)

---

## E4: Read the card and label

### US-09: Read a cited card for each paper

**Persona**: P1 · **Demo path** · **Depends on**: US-04
**Covers**: FR-4.1, FR-4.2, FR-4.3, FR-4.4, NFR-2.2

As a student researcher, I want a short card for each paper with its key details and findings, so that I can remember what it says without rereading it.

**Acceptance criteria**
- **Given** a paper has been indexed, **when** the card is written, **then** a fixed sequence of model calls is used (no tool calls chosen by the model), and no single prompt contains the whole paper collection
- **Given** the model's card output, **when** it is saved, **then** the card has: title, authors and year (each either a value or "unavailable"), findings relevant to the question each linked to at least one source passage, methods, limitations, and 1–2 quotes each with a PDF page number
- **Given** card output that is missing a required field or is badly formatted, **when** it is checked, **then** it is treated as a failed step (see US-08)
- [ ] The card shows the paper's display name: its title, or the original upload filename if the title is "unavailable"
- [ ] A card, once saved, is never changed by later uploads

### US-10: See which quotes were checked against the PDF

**Persona**: P1 · **Demo path** · **Depends on**: US-09
**Covers**: FR-5.1, FR-5.2, FR-5.3, FR-5.4, FR-5.5, FR-5.7, FR-11.5, FR-12.5, NFR-4.6

As a student researcher, I want to see which quotes were confirmed on their cited page, so that I know which quotes I can rely on and which I should check myself.

**Acceptance criteria**
- **Given** a quote from the model, **when** it is checked, **then** it is verified only if, after joining words split by line-break hyphens and collapsing whitespace, it has at least the minimum length (default 8 words) and appears on its cited PDF page
- **Given** a quote fails the check, **when** this is the first failure, **then** the model is asked once for a replacement quote, and the replacement is checked the same way
- **Given** a quote fails again, **when** the card is shown, **then** the quote is kept with the warning "Not verified: not found on PDF page N of [display name]", or "Not verified: quote too short to locate reliably" if it was too short
- **Given** no quote on a card is verified, **when** the card is saved, **then** it is still saved, and shows the card-level note "No quotes could be verified"
- [ ] Verified quotes, unverified quotes and the card-level note look clearly different from each other
- [ ] Nothing on the page says or suggests that a verified quote is interpreted correctly, only that it was found on that page

### US-11: Have my first relevant paper start the tracker

**Persona**: P1 · **Demo path** · **Depends on**: US-09
**Covers**: FR-6.0, FR-7.4, FR-8.2

As a student researcher, I want my first on-topic paper to start the first section, while an off-topic first paper is set aside, so that the tracker does not begin with a section about the wrong topic.

**Acceptance criteria**
- **Given** the tracker has no sections (empty, or holding only not-relevant papers), **when** a new paper's card is saved, **then** the comparison loop is skipped and one model call with no tools labels the paper either "New angle" or "Not relevant or relevance unclear"
- **Given** the label is "New angle", **when** the tracker is updated, **then** a first section is created with the model's proposed broad name, and the paper is placed in it
- **Given** either label, **when** it is shown, **then** it has a one-sentence reason citing at least one passage from the new paper only
- **Given** output that is not one of those two labels or lacks a reason or citation, **when** it is checked, **then** it is treated as a failed step (see US-08)
- [ ] The same written relevance criteria are used here and in the comparison loop (US-12)

### US-12: See how a new paper relates to what I've already read

**Persona**: P1 · **Demo path** · **Depends on**: US-11
**Covers**: FR-6.1, FR-6.2, FR-6.3, FR-6.4, FR-6.5, FR-6.6, FR-6.7, FR-7.1, FR-7.4, FR-7.5

As a student researcher, I want each new paper labeled against the papers already in my tracker, with a reason, so that I can see at a glance whether it opens a new angle or adds to something I have.

**Acceptance criteria**
- **Given** at least one section exists, **when** a new paper's card is saved, **then** a tool-calling loop starts, seeded with the most similar saved cards of papers that have a primary section, and existing cards are given to the model in small batches, never all at once
- **Given** the loop is running, **when** the model calls tools, **then** only the read-only tools (search passages, read a saved card, list sections) are available, they return only papers that have a primary section and their passages, and the loop stops at the tool-call cap
- **Given** the tracker also holds not-relevant papers, **when** the loop runs, **then** no seed, tool result or prompt contains a not-relevant paper's card or passages
- **Given** the cap is reached without a valid label, **when** the loop stops, **then** one final model call with tools turned off asks for a label from the evidence gathered; if that output is invalid it is a failed step (see US-08)
- **Given** a final label, **when** it is checked, **then** it is exactly one of "New angle", "Adds to [existing section]" with a relationship of agrees, contradicts, extends or unclear, "Redundant with [existing paper]", or "Not relevant or relevance unclear"
- **Given** an "Adds to" or "Redundant" label, **when** it is checked, **then** its reason cites at least one passage from the new paper and one from the other paper
- **Given** any label, **when** it is checked, **then** every cited passage came from a tool result or a seeded card; a citation to a passage the model never retrieved, or a label or citation that names a not-relevant paper, makes the output invalid (see US-08)
- [ ] "New angle" creates a new section with a broad name; "Adds to" places the paper in the named existing section
- [ ] The label text and reason describe the paper's place in the uploaded collection only, not its value or the state of the field

### US-13: Keep a redundant paper without losing it

**Persona**: P1 · **Depends on**: US-12
**Covers**: FR-7.1 (Redundant), FR-9.2

As a student researcher, I want a paper that repeats an earlier one to be marked as redundant but still kept, so that I can see the overlap without the paper disappearing.

**Acceptance criteria**
- **Given** a paper labeled "Redundant with [paper]", **when** the tracker updates, **then** the paper is placed in the referenced paper's section and its card is saved like any other
- [ ] The label names the earlier paper by display name and links to its card
- [ ] The label is explained as overlap within the uploaded collection, not a judgment of the paper's quality
- [ ] The paper counts toward its section, and the redundant count is also shown separately (see US-20)

### US-14: See off-topic papers set aside, not discarded

**Persona**: P1 · **Depends on**: US-11
**Covers**: FR-7.1 (Not relevant), FR-8.4, FR-9.3, FR-6.1 (exclusion), FR-6.2 (exclusion)

As a student researcher, I want papers that don't fit my question listed separately with a reason, so that they don't distort my sections but I can still see why they were set aside.

**Acceptance criteria**
- [ ] Papers labeled "Not relevant or relevance unclear" appear in their own group, separate from the sections
- [ ] Each shows its card and its one-sentence reason
- [ ] They are left out of section counts but included in the total paper count (see US-20)
- [ ] They are never placed in a section and are not offered as a section for later papers to add to
- [ ] They are never used as a comparison target: they are not seeds, not returned by the comparison tools, and never named or cited in another paper's label (see US-12)
- [ ] A new paper that resembles a not-relevant paper is judged on its own against the shared relevance criteria

### US-15: See other sections a paper relates to

**Persona**: P1 · **Depends on**: US-12
**Covers**: FR-7.2, FR-7.3

As a student researcher, I want a paper that touches several subtopics to show links to the other sections it relates to, so that I can see connections without the paper being counted twice.

**Acceptance criteria**
- [ ] Each relevant paper has exactly one primary section and zero or more secondary "related sections"
- [ ] Related sections are links only, chosen from sections that already exist when the paper is uploaded
- [ ] A paper's related sections never change after it is saved, even when later sections are created
- [ ] Related sections are not used in any count

---

## E5: Inspect evidence

### US-16: Expand a citation in place

**Persona**: P1 · **Demo path** · **Depends on**: US-09
**Covers**: FR-12.2, FR-5.6

As a student researcher, I want short citations that I can click to read the passage, so that the page stays readable but the evidence is one click away.

**Acceptance criteria**
- [ ] Each citation on the page is a short tag: first author's surname and year if both are known, otherwise the display name, plus "PDF p. N", for example "(Smith 2021, PDF p. 4)"
- [ ] Clicking a tag shows the cited passage in place, and clicking again hides it
- [ ] Page numbers are 1-based PDF page numbers, and the page explains that these may differ from the numbers printed in the paper

### US-17: Open the PDF at the cited page

**Persona**: P1 · **Demo path** · **Depends on**: US-09
**Covers**: FR-12.3, FR-12.4, A-5

As a student researcher, I want to open the original PDF at the page a claim cites, so that I can read the quote in its full context.

**Acceptance criteria**
- [ ] Each quote and each cited passage has a link that opens the stored PDF in the browser at the cited PDF page
- [ ] PDFs are served by paper ID only; a request containing a file path or a paper ID that does not exist returns "not found" and no file
- [ ] The cited passage text is always shown next to the link, so the evidence is readable even if the browser does not jump to the page

### US-18: See the evidence behind a label

**Persona**: P1 · **Demo path** · **Depends on**: US-11
**Covers**: FR-7.4, FR-12.3

As a student researcher, I want to see the passages a label is based on, so that I can decide for myself whether I agree with it.

**Acceptance criteria**
- **Given** any labeled paper, **when** the student opens its label details, **then** the page shows the label, its one-sentence reason, and each passage it relied on, with the paper's display name and PDF page
- **Given** an "Adds to" or "Redundant" label, **when** the details are shown, **then** passages from both papers are shown, grouped by paper
- [ ] Each passage has the PDF link from US-17

---

## E6: See the tracker and its coverage

### US-19: Read an overview that grows with my reading

**Persona**: P1 · **Demo path** · **Depends on**: US-11
**Covers**: FR-8.1, FR-8.2, FR-8.3, FR-8.5, FR-7.6, FR-12.1, NFR-6.3

As a student researcher, I want the tracker's overview and section summaries to update as I add papers, so that I always have a current picture of what my collection says.

**Acceptance criteria**
- **Given** a paper is classified, **when** the tracker updates, **then** its card is added, its section is created or updated, and the overview is rewritten
- **Given** a paper is added to an existing section, **when** the section summary is rewritten, **then** the section's name stays exactly the same, and every earlier card in the section is unchanged
- [ ] The tracker page shows the question, the overview, each section with its summary and count, each paper's card, label, reason and related sections, and the not-relevant group
- [ ] The page states that all assessments cover only the uploaded papers, not the research field
- [ ] The page offers no way to edit labels or sections, or to remove papers

### US-20: See how many papers each section has

**Persona**: P1 · **Demo path** · **Depends on**: US-19
**Covers**: FR-9.1, FR-9.2, FR-9.3, FR-9.5, SC-5

As a student researcher, I want to see paper counts per section, so that I can tell which subtopics are well covered and which are thin.

**Acceptance criteria**
- [ ] Each section shows the number of papers whose primary section it is
- [ ] Redundant papers count toward their section, and the total number of redundant papers is also shown
- [ ] Not-relevant papers are left out of section counts but included in the total paper count
- [ ] Counts are computed from saved data each time they are shown and always match the saved collection exactly

### US-21: See the publication-year range

**Persona**: P1 · **Demo path** · **Depends on**: US-09
**Covers**: FR-9.4, A-6, SC-5

As a student researcher, I want to see the earliest and latest publication years in my collection, so that I know whether my reading is recent or dated.

**Acceptance criteria**
- [ ] The tracker shows the earliest and latest known publication years across all saved papers, including not-relevant ones
- [ ] Papers whose year is "unavailable" are counted and shown separately, for example "2 papers with unknown year"
- [ ] With no known years, the range says so instead of showing a blank or a wrong value

---

## E7: Export and return later

### US-22: Export a readable summary

**Persona**: P1 · **Depends on**: US-19
**Covers**: FR-13.1, FR-13.2, FR-13.3, FR-13.4

As a student researcher, I want to download my tracker as a Markdown file, so that I can use it in my own notes and writing.

**Acceptance criteria**
- [ ] `tracker.md` is rewritten from saved data after every successful upload, with no model call
- [ ] It contains the question, the overview, the sections with summaries and counts, each card with its label and reason, the not-relevant group, the coverage counts and the year range
- [ ] Citations are numbered footnotes ([1], [2], …). Each gives the display name, the PDF page and the quoted passage, and marks unverified quotes as "Not verified"
- [ ] An "Export" action downloads the current `tracker.md`
- [ ] A failed upload leaves `tracker.md` exactly as it was

### US-23: Reopen my tracker and keep going

**Persona**: P1 · **Demo path** · **Depends on**: US-04
**Covers**: FR-1.3, FR-14.1, FR-14.2, FR-14.3, FR-4.4, NFR-3.2, SC-1, SC-4

As a student researcher, I want to close the app and come back another day to find everything as I left it, so that I can build my tracker over weeks without uploading papers again.

**Acceptance criteria**
- **Given** a tracker with saved papers, **when** the app is stopped and started again, **then** opening it goes straight to the tracker, with every card, section, label, reason, citation and quote status as before
- **Given** a reopened tracker, **when** the student opens a citation or a PDF link, **then** the passage and PDF are still available without re-uploading
- **Given** a reopened tracker, **when** the student adds another paper, **then** every earlier card and its citations are unchanged, and earlier sections keep their names; only section summaries and the overview may be rewritten
- [ ] Structured data is kept in SQLite, passages and embeddings in Chroma, and PDFs and `tracker.md` in the data folder

---

## E8: Evaluate the app

### US-24: Run the evaluation without touching my real tracker

**Persona**: P2 · **Depends on**: US-04, US-12
**Covers**: FR-17.1, FR-17.2, FR-17.4, FR-17.5

As a team evaluator, I want one command that runs the labeled test papers through the app in its own folder, so that I can measure the app without mixing test papers into a real tracker.

**Acceptance criteria**
- [ ] One command (for example `uv run python -m research_tracker.evaluate`) runs every test paper through the same processing pipeline as the web app, one at a time
- [ ] The run uses `eval-data/`, which is cleared at the start of each run; the `data/` folder is never read or changed
- [ ] The team's manual labels are read from a file in a format defined in design
- [ ] Setting the data folder in `.env` to `eval-data/` lets the evaluator open the run's results in the web page
- [ ] A paper that fails processing is reported as failed, and the run continues with the next paper

### US-25: See label agreement and count accuracy

**Persona**: P2 · **Depends on**: US-24
**Covers**: FR-17.3, SC-3, SC-5

As a team evaluator, I want a report comparing the app's labels and counts with ours, so that I know whether we meet the label and count targets.

**Acceptance criteria**
- [ ] The report shows, for each paper, the app's primary label and the team's label, and whether they match
- [ ] It shows the overall agreement rate and whether it meets the 80% target (SC-3)
- [ ] It checks that the section counts, redundant count, total count and year range shown by the app match values recomputed from the saved data, and lists any mismatch (SC-5)

### US-26: Confirm verified quotes by eye

**Persona**: P2 · **Depends on**: US-24, US-10
**Covers**: FR-17.6, SC-2

As a team evaluator, I want a checklist of every verified quote with a link to its page, so that a person, not the app's own code, confirms that verified quotes really are on their cited page.

**Acceptance criteria**
- [ ] The run writes a checklist to `eval-data/` listing every quote marked verified, each with the paper's display name, the quote, its PDF page and a link that opens the PDF at that page
- [ ] Each entry has a place to mark "found" or "not found"
- [ ] Running the script's checklist step on the completed file reports the share of verified quotes marked found, and whether it is 100% (SC-2)
- [ ] An entry left unmarked is reported as unmarked, not counted as found

### US-27: Catch mismatches between saved and shown quotes

**Persona**: P2 · **Depends on**: US-24, US-10
**Covers**: FR-17.7

As a team evaluator, I want the script to re-check quotes from what was actually saved and exported, so that wiring bugs between the check and the display are caught.

**Acceptance criteria**
- [ ] The script reads each quote, its PDF page and its verified flag from the saved database and from `tracker.md`, not from the pipeline's in-memory results
- [ ] It re-checks each quote against the stored page text
- [ ] It reports any quote whose flag, page or text differs between the database and `tracker.md`, or whose stored flag disagrees with the re-check

### US-28: See how long each step takes on my laptop

**Persona**: P2 · **Depends on**: US-04, US-24
**Covers**: FR-16.1, FR-16.2, FR-17.3 (timing), NFR-2.1

As a team evaluator, I want a timing summary for each processing step, so that we know which step is slow on which laptop before setting any speed target.

**Acceptance criteria**
- [ ] Every upload, in the web app or the evaluation, writes the time of each step to a local log file
- [ ] The evaluation report summarizes time per step (for example average and longest) and time per paper
- [ ] Logs never leave the laptop

### US-29: Compare settings fairly

**Persona**: P2 · **Depends on**: US-24
**Covers**: FR-15.1 (thinking mode), FR-3.3, NFR-2.2, A-2, A-3

As a team evaluator, I want to change a setting such as thinking mode or chunk size and re-run the evaluation, so that we can choose the setup that works best from data.

**Acceptance criteria**
- [ ] Thinking mode, model name and context length can be changed in `.env` without code changes; tuning values set in design (chunk size, batch size, tool-call cap, minimum quote length, "too little text" threshold) are kept in one settings place
- [ ] The report states the settings used for the run
- [ ] Because `eval-data/` is cleared at the start of each run, results from different settings never mix
- [ ] Turning thinking mode on or off changes how the model is called, as confirmed in design (A-3)

---

## INVEST Check

| Story | Independent (deps listed, testable alone, no loops) | Valuable | Small | Testable | Notes |
|---|---|---|---|---|---|
| US-01 | Yes | Yes | Yes | Yes | Tested by following the README on a clean laptop |
| US-02 | Yes | Yes | Yes | Yes | Settings loading tested with test `.env` values |
| US-03 | Yes | Yes | Yes | Yes | `TestClient` |
| US-04 | Yes | Yes | Yes | Yes | Fake model and fake embeddings; progress states tested directly |
| US-05 | Yes | Yes | Yes | Yes | Small inline sample PDFs |
| US-06 | Yes | Yes | Yes | Yes | Seed a saved paper hash |
| US-07 | Yes | Yes | Yes | Yes | Seed a "processing" state |
| US-08 | Yes | Yes | Yes | Yes | Fake model returning bad output; seed an interrupted record |
| US-09 | Yes | Yes | Yes | Yes | Fake model card output |
| US-10 | Yes | Yes | Yes | Yes | Pure functions plus fake replacement quote |
| US-11 | Yes | Yes | Yes | Yes | Seed an empty tracker or one with only not-relevant papers |
| US-12 | Yes | Yes | Medium | Yes | Largest story; kept whole because the loop's rules only make sense together. Seed sections and cards |
| US-13 | Yes | Yes | Yes | Yes | Fake model returning a redundant label |
| US-14 | Yes | Yes | Yes | Yes | Seed a not-relevant paper |
| US-15 | Yes | Yes | Yes | Yes | Seed sections, then add one later |
| US-16 | Yes | Yes | Yes | Yes | Seed a card |
| US-17 | Yes | Yes | Yes | Yes | Seed a stored PDF |
| US-18 | Yes | Yes | Yes | Yes | Seed a label with passages |
| US-19 | Yes | Yes | Yes | Yes | Seed a section, add a paper with the fake model |
| US-20 | Yes | Yes | Yes | Yes | Pure count function plus seeded data |
| US-21 | Yes | Yes | Yes | Yes | Pure function |
| US-22 | Yes | Yes | Yes | Yes | Render from seeded data |
| US-23 | Yes | Yes | Yes | Yes | Seed data, restart the app in tests |
| US-24 | Yes | Yes | Yes | Yes | Fake model and a two-paper test set |
| US-25 | Yes | Yes | Yes | Yes | Seeded results plus a label file |
| US-26 | Yes | Yes | Yes | Yes | Seeded verified quotes, completed checklist |
| US-27 | Yes | Yes | Yes | Yes | Seed a deliberate mismatch |
| US-28 | Yes | Yes | Yes | Yes | Seeded log file |
| US-29 | Yes | Yes | Yes | Yes | Two runs with different settings |

All stories are **Negotiable** (details can change in design) and **Estimable** (scope is bounded by the listed requirement IDs).

**Dependency levels** (checked: no loops). A story depends only on stories at earlier levels:

| Level | Stories |
|---|---|
| 0 | US-01 |
| 1 | US-02, US-03 |
| 2 | US-04, US-05 |
| 3 | US-06, US-07, US-08, US-09, US-23 |
| 4 | US-10, US-11, US-16, US-17, US-21 |
| 5 | US-12, US-14, US-18, US-19 |
| 6 | US-13, US-15, US-20, US-22, US-24 |
| 7 | US-25, US-26, US-27, US-28, US-29 |

**Demo path** (SC-1), in order: US-01, US-03, US-04, US-09, US-10, US-11, US-12, US-16, US-17, US-18, US-19, US-20, US-21, US-23.

---

## Requirements Coverage

Each row points to a story, a shared constraint (C-n), or **Team practice** (checked in Build and Test).

### Functional requirements

| ID | Covered by | ID | Covered by |
|---|---|---|---|
| FR-1.1 | US-03 | FR-8.1 | US-19 |
| FR-1.2 | US-03 | FR-8.2 | US-11, US-19 |
| FR-1.3 | US-23 | FR-8.3 | US-19 |
| FR-1.4 | US-03 | FR-8.4 | US-14 |
| FR-2.1 | US-04 | FR-8.5 | US-19 |
| FR-2.2 | US-05 | FR-9.1 | US-20 |
| FR-2.3 | US-06 | FR-9.2 | US-13, US-20 |
| FR-2.4 | US-04 | FR-9.3 | US-14, US-20 |
| FR-2.5 | US-07 | FR-9.4 | US-21 |
| FR-3.1 | US-04 | FR-9.5 | US-20, C-7 |
| FR-3.2 | US-04 | FR-10.1 | US-04 |
| FR-3.3 | US-04, US-29 | FR-10.2 | US-04 |
| FR-3.4 | US-04 | FR-10.3 | US-07 |
| FR-4.1 | US-09 | FR-11.1 | US-08 |
| FR-4.2 | US-09 | FR-11.2 | US-08 |
| FR-4.3 | US-09 | FR-11.3 | US-08 |
| FR-4.4 | US-09, US-23 | FR-11.4 | US-08 |
| FR-5.1 | US-10, C-7 | FR-11.5 | US-10 |
| FR-5.2 | US-10 | FR-11.6 | US-08 |
| FR-5.3 | US-10 | FR-12.1 | US-19 |
| FR-5.4 | US-10 | FR-12.2 | US-16 |
| FR-5.5 | US-10 | FR-12.3 | US-17, US-18 |
| FR-5.6 | US-16 | FR-12.4 | US-17 |
| FR-5.7 | US-10 | FR-12.5 | US-10 |
| FR-6.0 | US-11 | FR-13.1 | US-22, C-7 |
| FR-6.1 | US-12, US-14 | FR-13.2 | US-22 |
| FR-6.2 | US-12, US-14, C-3 | FR-13.3 | US-22 |
| FR-6.3 | US-12 | FR-13.4 | US-22 |
| FR-6.4 | US-12 | FR-14.1 | US-23 |
| FR-6.5 | US-12 | FR-14.2 | US-23 |
| FR-6.6 | US-12 | FR-14.3 | US-23 |
| FR-6.7 | US-12 | FR-15.1 | US-02, US-29 |
| FR-7.1 | US-12, US-13, US-14 | FR-15.2 | US-01 |
| FR-7.2 | US-15 | FR-15.3 | US-02 |
| FR-7.3 | US-15 | FR-15.4 | US-01 |
| FR-7.4 | US-11, US-12, US-18 | FR-16.1 | US-04, US-28 |
| FR-7.5 | US-12 | FR-16.2 | US-28 |
| FR-7.6 | US-19 | FR-17.1 | US-24 |
| | | FR-17.2 | US-24 |
| | | FR-17.3 | US-25, US-28 |
| | | FR-17.4 | US-24 |
| | | FR-17.5 | US-24 |
| | | FR-17.6 | US-26 |
| | | FR-17.7 | US-27 |

### Non-functional requirements

| ID | Covered by |
|---|---|
| NFR-1.1 | C-1 |
| NFR-1.2 | C-2 |
| NFR-1.3 | C-3 |
| NFR-1.4 | C-4 |
| NFR-1.5 | C-5 |
| NFR-1.6 | Team practice (`.gitignore` contents) |
| NFR-1.7 | US-05 |
| NFR-2.1 | US-28 |
| NFR-2.2 | US-09, US-29 |
| NFR-3.1 | US-08 |
| NFR-3.2 | US-23 |
| NFR-3.3 | Team practice (no pickle or shelve) |
| NFR-4.1 | Team practice (pytest; tests pass before merge) |
| NFR-4.2 | Team practice (fake model, `TestClient`) |
| NFR-4.3 | Team practice (model tests behind a marker) |
| NFR-4.4 | Team practice (property-based tests, PBT Partial) |
| NFR-4.5 | Team practice |
| NFR-4.6 | US-10, Team practice (hard-case unit tests) |
| NFR-5.1 | Team practice (prohibited libraries) |
| NFR-5.2 | Team practice (plain Python prompts and loop) |
| NFR-5.3 | US-01, Team practice (Python 3.12, `uv.lock` committed) |
| NFR-6.1 | US-05, C-6 |
| NFR-6.2 | US-04 |
| NFR-6.3 | US-19 |

### Success criteria and assumptions

| ID | Covered by |
|---|---|
| SC-1 | Demo path stories, ending in US-23 |
| SC-2 | US-26 (supported by US-27 and US-10) |
| SC-3 | US-25 |
| SC-4 | US-23 |
| SC-5 | US-20, US-21, US-25 |
| A-1 | US-01, US-05 |
| A-2 | US-29 |
| A-3 | US-29 |
| A-4 | US-02 |
| A-5 | US-17 |
| A-6 | US-21 |
