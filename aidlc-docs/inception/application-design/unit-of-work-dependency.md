# Unit Dependencies: Research Tracker

## Dependency Matrix

Read across: the row unit **uses** something owned by the column unit. "I" means it uses only an **interface** defined in U0, with a fake available, so the row unit never waits for the column unit's code. "C" means it calls the column unit's real code at run time. A fake or stub stands in during tests.

| Uses → | U0 Foundation | U1 PDF and Retrieval | U2 Paper Cards | U3 Comparison | U4 Summaries and Integration | U5 Interface and Persistence |
|---|---|---|---|---|---|---|
| **U0** | — | | | | | |
| **U1** | C | — | | | | I (`TrackerStore`) |
| **U2** | C | | — | | | |
| **U3** | C | C (`InMemoryPassages`, `embed_text`) | | — | | I (`TrackerStore`) |
| **U4** | C | C (`check_upload`, Indexer) | C (`write_card`, `check_quote`) | C (`classify`) | — | I (`TrackerStore`) |
| **U5** | C | C (`PassageIndex` inside `TrackerStore`; `check_upload` in the upload route) | | | C (`compute_coverage`, `render_markdown`, `JobRunner`) | — |

### Are there loops?

- **At the component level, no.** `component-dependency.md` shows six layers, and a script confirmed there are no upward dependencies.
- **At the unit level, two pairs of units use each other.** Units group components by owner, not by layer:
  - **U1 ↔ U5**:
    - U1's Intake asks `TrackerStore` about duplicates and the paper count
    - U5's `TrackerStore` calls U1's `passages.py`
  - **U4 ↔ U5**:
    - U4's Processor and Job Runner save through `TrackerStore`
    - U5's Web calls the Job Runner, and `TrackerStore` uses U4's `compute_coverage` and `render_markdown`
- **Why these pairs don't block anyone**:
  - In each pair, one direction goes through an interface frozen in U0 (`TrackerStore` or `PassageIndex`), and a fake stands in for it.
  - The other direction uses pure functions (`compute_coverage`, `render_markdown`) or a narrow API (`JobRunner`), with signatures fixed in `component-methods.md`.
  - Neither unit waits for the other's code while building.
  - The real code meets in the end-to-end test, which U4 keeps green.

## Contracts

These are the interfaces units depend on. A change to a contract needs approval from its owner and every unit listed as a user, and is recorded in the main `audit.md`.

| Contract | Defined in | Owned after freeze | Used by | Signatures in |
|---|---|---|---|---|
| Shared data types (`models.py`) | U0 | Shared (all affected units review) | All | components.md |
| `Settings`, `load_settings` | U0 | U5 | All | component-methods.md §1 |
| `ModelClient`, `ModelReply`, `ModelError`, `FakeModelClient` | U0 | U2 | U1, U2, U3, U4 | §4 |
| `TrackerStore` interface, `FakeTrackerStore`, contract tests | U0 | U5 | U1, U3, U4, U5 (Web) | §12 |
| `PassageIndex` interface (Chroma) | U0 | U1 | U5 (`TrackerStore` only) | §12 (Chroma part) |
| `check_upload`, `PdfRejected` | U1 | U1 | U4 (Evaluation), U5 (upload route) | §2 |
| `chunk_pages`, `embed_chunks`, `embed_text`, `InMemoryPassages` | U1 | U1 | U3, U4 | §3 |
| `write_card` | U2 | U2 | U4 | §5 |
| `normalize`, `check_quote`, `check_with_replacement` | U2 | U2 | U2, U4 (Evaluation) | §6 |
| `classify`, `ComparisonFailed` | U3 | U3 | U4 | §7 |
| Tool contract (`search_passages`, `read_card`, `list_sections`) | U3, designed first | U3 | U3, with result data supplied through `TrackerStore` (U5) | §8 |
| `plan_update` | U4 | U4 | U4 | §9 |
| `compute_coverage` | U4 | U4 | U5, U4 (Evaluation) | §10 |
| `render_markdown` | U4 | U4 | U5 (`write_tracker_md`), U4 (Evaluation) | §11 |
| `PaperProcessor.process`, `ProgressCallback`, `ProcessingFailed` | U0 skeleton | U4 | U4 (Job Runner, Evaluation) | §13 |
| `JobRunner` (`submit`, `status`, `Busy`) | U0 skeleton | U4 | U5 (Web) | §14 |
| SQLite schema | U5, designed first | U5 | U5 only (other units go through `TrackerStore`) | Functional Design U5 |
| HTTP routes | U5 | U5 | Browser, `poll.js` | §15 |
| Label file format | U4 with the test set workstream | U4 | Test set workstream, Evaluation | Functional Design U4 |

## Build Order and Parallel Work

The team chooses a track by **Nov 25** (unit plan follow-up 2).

```text
Phase 1  Foundation (U0)
           Track C: volunteers build Nov 25-29; whole team reviews and freezes on Nov 30
           Track B: whole team builds Nov 30 - Dec 1 and freezes
         Meanwhile (both tracks): test-set paper selection (workstream, due Nov 27)

Phase 2  Parallel units (start Dec 1 on Track C, Dec 2 on Track B)
           First, before other Functional Designs depend on them:
             U5 Functional Design: SQLite schema
             U3 Functional Design: tool contract result fields
           Then all in parallel:
             U1, U2, U3, U4, U5 Functional Design -> Code Generation
           Continuously: U4 keeps the end-to-end fake-model test green;
             U5's real TrackerStore replaces the fake once it passes the contract tests
           Test set workstream: label file format (during U4 Functional Design),
             optional fake-model data from two papers (Nov 30), independent labels (Dec 3),
             frozen label file (Dec 5)

Phase 3  Dec 9-13: integration checks with real Ollama, evaluation runs, tuning
           (thinking mode, chunk size, cap), Build and Test stage

Phase 4  Dec 14-16: manual quote checklist (SC-2), demo rehearsal (SC-1), fixes, buffer
```

| Unit | Can start Construction when | Must finish Functional Design before |
|---|---|---|
| U0 | Units Generation approved | Phase 2 starts |
| U1 | U0 frozen | — |
| U2 | U0 frozen | — |
| U3 | U0 frozen | Other units rely on the tool result fields (first days of Phase 2) |
| U4 | U0 frozen. Its end-to-end test uses stubs until U1–U3 land | Label file format is due during its Functional Design (test set dates) |
| U5 | U0 frozen | Other units rely on the schema (first days of Phase 2) |

### Text Alternative: Unit Dependencies

```text
U0 Foundation  <- used by every unit (types, Settings, Model Client, interfaces, fakes)
U1 PDF/Retrieval -> U0; TrackerStore interface (U5)
U2 Paper Cards   -> U0
U3 Comparison    -> U0; U1 (InMemoryPassages, embed_text); TrackerStore interface (U5)
U4 Summaries/Integration -> U0; U1; U2; U3; TrackerStore interface (U5)
U5 Interface/Persistence -> U0; U1 (PassageIndex, check_upload); U4 (coverage, export, JobRunner)
Mutual pairs (U1-U5, U4-U5) cross only frozen interfaces or pure functions; no build-time waiting.
```
