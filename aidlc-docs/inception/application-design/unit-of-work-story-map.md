# Unit-to-Story Map: Research Tracker

Every story has exactly one **primary** unit, which is accountable for its acceptance criteria and owns its main tests. **Supporting** units build parts the story needs, and their owners review the story's tests.

## Story Map

| Story | Title | Primary unit | Supporting units | Demo path |
|---|---|---|---|---|
| US-01 | Install and start the app | U5 | U0 (project setup, Modelfile, README first version) | Yes |
| US-02 | Switch to a smaller model | U5 (Settings) | U2 (Model Client uses the setting) | |
| US-03 | Create a tracker | U5 | | Yes |
| US-04 | Upload a paper and watch it being processed | U4 (Processor, Job Runner, progress, timing) | U1 (extraction, indexing), U5 (upload route, status route, polling) | Yes |
| US-05 | Clear reason when upload is rejected | U1 | U5 (error display) | |
| US-06 | Stopped from adding the same PDF twice | U1 | U5 (`paper_exists`) | |
| US-07 | Told to wait while a paper is processing | U4 (Job Runner `Busy`) | U5 (server response, disabled control) | |
| US-08 | Try again when processing fails | U4 (retries, discard, start-up recovery) | U5 (`commit`, `cleanup_uncommitted`, notice display and dismissal) | |
| US-09 | Read a cited card | U2 | U5 (card display) | Yes |
| US-10 | See which quotes were checked | U2 | U5 (status display) | Yes |
| US-11 | First relevant paper starts the tracker | U3 | U4 (first section via `plan_update`) | Yes |
| US-12 | See how a new paper relates | U3 | U5 (`TrackerStore` relevant-only reads), U1 (`InMemoryPassages`) | Yes |
| US-13 | Keep a redundant paper | U3 | U4 (placement and redundant count), U5 (display) | |
| US-14 | Off-topic papers set aside | U3 | U4 (counts), U5 (not-relevant group display) | |
| US-15 | See related sections | U3 | U5 (links display) | |
| US-16 | Expand a citation in place | U5 | | Yes |
| US-17 | Open the PDF at the cited page | U5 | | Yes |
| US-18 | See the evidence behind a label | U5 | U3 (citations on the label) | Yes |
| US-19 | Overview that grows with my reading | U4 (Tracker Updater) | U5 (tracker page) | Yes |
| US-20 | Paper counts per section | U4 (Coverage) | U5 (display) | Yes |
| US-21 | Publication-year range | U4 (Coverage) | U5 (display) | Yes |
| US-22 | Export a readable summary | U4 (Exporter) | U5 (`write_tracker_md`, export route) | |
| US-23 | Reopen my tracker and keep going | U5 | U4 (start-up recovery) | Yes |
| US-24 | Run the evaluation separately | U4 | U1 (`check_upload`) | |
| US-25 | Label agreement and count accuracy | U4 | | |
| US-26 | Confirm verified quotes by eye | U4 (may move to U2, see the workload note) | U2 (quote statuses) | |
| US-27 | Catch saved-versus-shown quote mismatches | U4 (may move to U2) | U2 (`check_quote`) | |
| US-28 | Step timing on my laptop | U4 | | |
| US-29 | Compare settings fairly | U4 | U5 (Settings), U2 (thinking mode in the Model Client) | |

## Stories per Unit

| Unit | Primary stories | Count | Demo path stories |
|---|---|---|---|
| U0 Foundation | none (enables every story) | 0 | — |
| U1 PDF and Retrieval | US-05, US-06 | 2 | — |
| U2 Paper Cards | US-09, US-10 | 2 | US-09, US-10 |
| U3 Comparison | US-11, US-12, US-13, US-14, US-15 | 5 | US-11, US-12 |
| U4 Summaries and Integration | US-04, US-07, US-08, US-19, US-20, US-21, US-22, US-24, US-25, US-26, US-27, US-28, US-29 | 13 | US-04, US-19, US-20, US-21 |
| U5 Interface and Persistence | US-01, US-02, US-03, US-16, US-17, US-18, US-23 | 7 | US-01, US-03, US-16, US-17, US-18, US-23 |

The counts are uneven, but the effort is less so:
- U1's two stories include all upload checks, chunking, embeddings and the Chroma rules.
- U2's two stories include the whole card pipeline, quote checking and the Model Client.
- Five of U4's thirteen stories (US-20, US-21, US-22, US-25, US-28) are pure functions or summaries of saved data.

## Shared Constraints by Unit

| Constraint | Units responsible |
|---|---|
| C-1 Localhost only, telemetry off | U5 (Uvicorn host, README), U1 (Chroma telemetry) |
| C-2 No outside requests | U0, then U2 (Model Client), U5 (Settings: localhost URL check), U1 (Chroma never embeds) |
| C-3 Paper text never instructions | U2, U3, U4 (each owns its prompts). U3 reviews the others' prompts for consistency |
| C-4 Autoescaping | U5 |
| C-5 SQL placeholders | U5 |
| C-6 Plain error messages | U1 (rejections), U4 (processing failures, busy), U5 (web mapping, settings, schema mismatch) |
| C-7 Code, not model, for counts, quote checks, export | U2 (quotes), U4 (coverage, export) |

## "Team Practice" Requirements by Unit

| Requirement | Set up by | Kept by |
|---|---|---|
| NFR-1.6 `.gitignore` (`.env`, `data/`, `eval-data/`) | U0 | U5 |
| NFR-3.3 No pickle or shelve | — | U5 (storage), all reviewers |
| NFR-4.1 pytest; tests pass before merge | U0 | All (each unit's reviewer checks before merging) |
| NFR-4.2 Fake model, `TestClient` | U0 | All |
| NFR-4.3 Model tests behind a marker | U0 | All |
| NFR-4.4 Property-based tests (PBT Partial) | U0 (Hypothesis set up; round trips for `models.py`) | U1 (`chunk_pages`), U2 (`normalize`, `check_quote`), U4 (`compute_coverage`, `render_markdown`) |
| NFR-4.5 Model behavior via fake-model tests and evaluation | — | U4 (Evaluation), U2, U3 (fake-model tests) |
| NFR-4.6 Quote hard-case tests | — | U2 |
| NFR-5.1 Prohibited libraries | U0 (dependency list) | All reviewers |
| NFR-5.2 Plain Python prompts and loop | — | U2, U3, U4 |
| NFR-5.3 Python 3.12, `uv.lock` committed | U0 | U5 |

## Check

- Every story (US-01 to US-29) has exactly one primary unit. This was checked by script; see the stage completion log in `audit.md`.
- Every component in `components.md` belongs to exactly one unit:
  - U0 creates the shared types, interfaces, fakes and skeletons, then hands them over as listed in `unit-of-work.md`
  - U1: PDF Intake, Indexer, `passages.py`
  - U2: Card Writer, Quote Checker, and the Model Client after handover
  - U3: Comparison Agent, Agent Tools
  - U4: Tracker Updater, Coverage, Exporter, Processor, Job Runner, timing, Evaluation
  - U5: Web, the rest of Storage, and Settings after handover
