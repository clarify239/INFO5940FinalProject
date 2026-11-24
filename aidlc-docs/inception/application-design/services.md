# Services: Research Tracker

There are three services, meaning components that call other components in sequence. The Paper Processor is the only one that chains processing steps (Q1). The Job Runner decides *when* the Processor runs in the app, and the Evaluation decides when it runs in tests.

## 1. Paper Processor

**Owns**: one paper's journey from `ExtractedPdf` to a committed `PaperRecord`, or to nothing.

### Flow

```text
(upload request, in Web)
  check_upload -> ExtractedPdf                    # FR-2.2, rejections are immediate
  JobRunner.submit                                # saves the job record, or raises Busy

(background, in the Processor)
  1. indexing          chunk_pages, embed_chunks
  2. writing_card      write_card (includes quote checks and one replacement per failed quote)
                       embed_text(card) -> card_embedding
  3. comparing         classify (first-paper call, or loop + final call)
  4. updating_tracker  plan_update (section summary + overview)
  -- everything so far lives in a PaperDraft in memory; nothing is on disk --
  5. commit            TrackerStore.commit(draft)      # PDF file + Chroma, then one SQLite transaction
  6. export            TrackerStore.write_tracker_md   # regenerated from saved data
  finish_job, report done
```

### Rules the Processor applies

| Concern | Behavior | Source |
|---|---|---|
| Progress | Calls `on_progress(step)` at the start of steps 1–4 | FR-10.2 |
| Timing | Times extraction (passed in from the request) and steps 1–4, and appends one record per step to the timing log, with the step name, duration, model name and settings | FR-16.1, CQ7 |
| Step retries | Steps 1, 2 and 4: a `StepFailed` or `ModelError` reruns that step, up to 2 more times | FR-11.1 |
| Comparison | Step 3 is never rerun as a whole. Retries happen inside the agent (turn re-asks, then retries of the final call only) | FR-6.3, FR-6.4, FR-11.1 |
| Failure | After the retries run out, the draft is discarded. Because nothing was written yet, nothing is left. `finish_job` runs, and `ProcessingFailed(step, message)` is raised | FR-11.2, FR-11.3 |
| Commit failure | If the commit itself fails partway (for example a disk error after the PDF is written), anything written is removed, by the same rule start-up cleanup uses: anything without a committed SQLite row is deleted | FR-11.2, Q4 |
| Export after commit | If writing `tracker.md` fails after a successful commit, the paper stays saved. `tracker.md` is regenerated at the next start-up or after the next upload | Q4 follow-on |
| Quote failures | Never fail a step. They become quote statuses and the card-level note | FR-5.4, FR-11.5 |

### Worst-case model calls per paper

Per attempt: card writing calls, plus at most one replacement call per quote. Then the comparison step, at most cap + 3 model calls. Then the summary and overview calls. Steps 1, 2 and 4 can each run up to 3 times. The exact numbers depend on the tuning values set in Functional Design.

## 2. Job Runner

**Owns**: running one paper at a time in the background, plus the start-up recovery.

### Submit and run

```text
submit(tracker_id, extracted)
  if busy -> raise Busy                      # FR-2.5, checked under a lock
  job_id = store.start_job(...)              # saved BEFORE processing, outside the paper commit (FR-11.6)
  status = processing(step=None, filename)
  hand (job_id, tracker_id, extracted) to the worker thread

worker thread
  processor.process(..., on_progress=update status.step)
  success -> status = done(paper_id)
  ProcessingFailed -> status = failed(message)
  finally -> store.finish_job(job_id); busy = False
```

- **One worker thread**, started with the app (Q2: B). It uses its own SQLite connection.
- **Status** is kept in memory and read by `GET /status`. A refreshed page or a new tab sees the current step right away (Q3).
- **After `done` or `failed`**, the status stays visible until the next upload, so the page can show the result or the error.

### Start-up recovery (before the worker starts)

```text
notices = store.cleanup_uncommitted()
  - delete PDF files and Chroma entries whose paper is not committed
  - every leftover job record (the app stopped mid-processing) becomes a notice
store.write_tracker_md(tracker)               # always regenerated; cheap, no model
JobRunner.start()
```

- **Notices** are shown on the tracker page until they are dismissed. Dismissing one deletes the record, so nothing about that paper remains (FR-11.6).

### Shutdown

`stop(timeout)` asks the worker to stop. Nothing partial can be committed, because commit is the last step. Anything already written is cleaned up at the next start.

## 3. Evaluation

**Owns**: measuring the pipeline on the labeled test set (Q9: A).

```text
run(labels, papers_dir)
  settings = load_settings(overrides={"data_dir": "eval-data"})
  delete and recreate eval-data/                          # FR-17.2
  store = TrackerStore(settings); tracker = create_tracker(question from label file)
  processor = PaperProcessor(store, client, settings)     # same Processor, no Job Runner
  for paper in labels (in upload order):
      extracted = check_upload(...)                      # same intake checks
      try processor.process(...) except ProcessingFailed -> record failure, continue
  report = label_agreement + count_check + stored_quote_check + timing_summary + settings used
  write_checklist(eval-data/quote-checklist.md)
  print and save the report

score-checklist(path)
  read found / not found / unmarked marks -> SC-2 result
```

- **Real model by default**, so the results measure Qwen3. Its own tests use the fake model.
- **Viewing results**: setting `DATA_DIR=eval-data` in `.env` and starting the web app opens the run's results (FR-17.4).

## Interaction Summary

| Caller | Calls | When |
|---|---|---|
| Web (upload route) | PDF Intake, then Job Runner `submit` | During the upload request |
| Web (status route) | Job Runner `status` | Every few seconds while the page polls |
| Web (pages, export, PDF) | TrackerStore reads, Coverage, Exporter | On page load or download |
| Job Runner | Paper Processor | In the worker thread |
| Paper Processor | Indexer, Card Writer, Comparison Agent, Tracker Updater, TrackerStore | In order, per paper |
| Comparison Agent | Agent Tools, Model Client | Inside the comparing step |
| Evaluation | PDF Intake, Paper Processor, TrackerStore, Coverage, Quote Checker, Exporter | Command line |
