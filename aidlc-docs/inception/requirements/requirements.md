# Research Tracker requirements

## Status and intent

Prepared for user review. Requirements generation is complete, but approval is pending. Later Inception stages have not been executed.

- Request type is a new greenfield project.
- Scope covers PDF ingestion, cited cards, agent comparison, summaries, persistence, and a local web interface.
- Complexity is moderate to complex because model output and evidence retrieval require validation.
- Requirements depth is standard, with explicit acceptance criteria for evidence and persistence.
- Inputs are README.md, VisionDocument.md, TechnicalDocument.md, the answered requirements questionnaire, and decisions accepted in chat.
- Nicole Lin, Monisha Bommu, Claire Cheng, and Amber Wang are the four listed team members. No fifth member will be invented or added.
- This milestone ends at approved Inception artifacts. No application code or actual model-quality results are claimed.

## Product boundaries

One local user manages one tracker with one research question, up to 10 saved papers and 20 PDF pages per paper. The interface uses FastAPI and Jinja2. SQLite is authoritative for tracker records, with local PDFs and extracted passages for evidence and Chroma for derived passage embeddings. Qwen3 8B is the initial model through Ollama's OpenAI-compatible API. A local `nomic-embed-text` model supplies embeddings. Dependency and model downloads require internet during setup. Runtime paper processing must use only local services.

## Functional requirements

| ID | Requirement and acceptance criteria |
|---|---|
| FR-01 | Create the sole tracker from a nonempty research question. On subsequent app starts, reopen the existing tracker instead of creating a new one. Initial summary and sections are empty. Changing the question and managing multiple trackers are out of scope. |
| FR-02 | Accept one English text-based PDF at a time, at most 20 PDF pages and 25 MiB. Reject encrypted, unparseable, image-only, empty-text, or over-limit uploads with an understandable message. A header check is only a preliminary filter and must be followed by PDF parsing. English is the supported input scope, not a promise of perfect automatic language detection. |
| FR-03 | Compute a SHA-256 digest before model processing. An identical saved PDF is rejected as a duplicate without invoking the model. Detecting revised or differently encoded copies is out of scope. Reject additions once 10 papers have been saved, including off-topic records. |
| FR-04 | Extract text with pypdf and preserve 1-based PDF page references through chunking. Split text into bounded chunks without silently truncating the document. If the document cannot be processed within configured limits, fail clearly rather than claiming it was fully processed. Tables, figures, and equations are not promised full interpretation. |
| FR-05 | A fixed card-generation pipeline processes the new paper in chunks and consolidates findings relevant to the question. Store title, authors, year, methods, findings, limitations, and one or two supporting quotes when available. Missing or uncertain metadata is unavailable. Do not invent metadata or fabricate quotes to reach the quote count. Insufficient evidence is explicitly reported. |
| FR-06 | Check nonempty quotations against their cited extracted page after whitespace and line-break hyphenation normalization. Mark matching quotes as location verified. Keep unmatched quotes visible with an unverified warning. Out-of-range references fail verification. This checks source location, not meaning or completeness of extraction. |
| FR-07 | Give the comparison agent allowlisted read tools for listing sections, reading saved cards, and retrieving local passages. It decides which additional passages to inspect within a finite configured tool-call and token budget. Compare the new card with every existing card in bounded batches. Never require the entire PDF collection in one context window. |
| FR-08 | Classify each relevant paper as new angle, adds to an existing section, or redundant with an identified earlier paper. A new angle creates a section. Every relevant paper has exactly one primary section. Additional independent data, a new population, method, or condition normally constitutes an addition. Redundant means no substantial new relevant evidence beyond the referenced paper, not a judgment of scientific worth. |
| FR-09 | An addition carries an agrees, contradicts, extends, or unclear relationship and identifies the earlier evidence being compared. Agreement or contradiction requires comparable claims. Explain differences in populations, methods, and conditions where relevant. Use unclear rather than force an unsupported conclusion. |
| FR-10 | Every classification includes a one-sentence reason and source passage references. Cross-paper relationship claims cite evidence from both papers. Store off-topic or relevance-unclear papers separately with a reason, without creating a misleading topical section. |
| FR-11 | After successful processing, add the new immutable card and citation records, update the affected section summary, and update the short overview. Section names stay fixed after creation. Do not rewrite earlier paper cards or remove prior membership. Summary claims reference stored evidence. |
| FR-12 | Compute counts in code from saved records. Each relevant paper counts once in its primary section, including redundant papers. Show the redundant subtotal separately. Off-topic records appear in total saved papers but not section totals. Show the earliest and latest known years across the saved collection and the number with unavailable years. An empty or all-unknown collection has no invented year range. |
| FR-13 | Persist cards, sections, classifications, quotes, PDF references, and extracted passages across app restarts without re-uploading. Chroma is a rebuildable derived index, not the authoritative record. Failed processing must not expose partial new records or modify the prior valid tracker. |
| FR-14 | The local page supports tracker creation, PDF upload, a processing indicator, section and card viewing, and evidence inspection. Citations are numbered within each card or summary, with paper identity, PDF page, passage, and verification status. Showing passage text is sufficient; an embedded PDF viewer is not required. |
| FR-15 | Upload waits for processing to finish while the page displays a processing indicator. Only one processing operation is accepted at a time. Other upload attempts receive a clear busy response. Step-by-step background jobs are out of scope. A bounded timeout or exhausted budget produces a recoverable failure. |
| FR-16 | Retry a failed model step up to two times after its initial attempt. Validate output shape, referenced IDs, labels, and page ranges before accepting changes. After persistent failure, preserve the previous tracker and show an error. Model-declared uncertainty remains visible rather than treated as a formatting failure. |
| FR-17 | Export tracker.md on demand from saved records. Include the question, overview, sections, cards, evidence references, and counts. Export failure does not corrupt saved records. Manual Markdown edits are not imported. |
| FR-18 | Provide an explicit evaluation command or script using a fixed-order collection of up to 10 manually labeled papers. Report primary-label agreement, quote-verification results, and count accuracy. Evaluation must use a separate temporary tracker and never change the user's tracker. Report the denominator and failures. |

## Nonfunctional requirements

| ID | Requirement and validation |
|---|---|
| NFR-01 | Offline runtime after dependencies and both local models have been installed. Bind app and model to loopback, disable Chroma telemetry, and do not invoke hosted model or search APIs. Test with external network access disabled. |
| NFR-02 | Treat paper text and model output as untrusted data. The LLM has no arbitrary filesystem, shell, SQL, URL-fetching, or internet tools. Application-controlled validated writes occur only after processing succeeds. Render text with HTML escaping and use parameterized database statements. |
| NFR-03 | Enforce upload byte and page limits, generated storage paths, and local tracker boundaries. Handle parser failures cleanly. PDF magic bytes alone do not prove a safe or valid PDF. |
| NFR-04 | Save canonical records transactionally and retain the previous valid tracker on failure. PDFs and the derived vector index require staging/cleanup or rebuild behavior because a SQLite transaction does not atomically cover all stores. Verify by injecting failures at save boundaries. |
| NFR-05 | Model name, embedding model, context length, chunk size, comparison batch size, thinking mode, timeouts, and tool budgets are explicit settings. Select initial values by measuring a representative paper during Construction. Enforce token budgets for instructions, evidence, tool results, and generated output. |
| NFR-06 | No promised runtime target before measurement. Record model tag, hardware, context settings, paper size, and end-to-end processing times. Smaller local models and reduced chunk/batch sizes are permitted configuration fallbacks, subject to repeated quality evaluation. Rebuild embeddings if the embedding model changes. |
| NFR-07 | Use Python 3.12, uv, FastAPI, Uvicorn, Jinja2, pypdf, SQLite, Chroma, and the OpenAI Python client pointed at Ollama. Follow the prohibited-library list. Declare and pin dependencies during Construction. No application dependency setup is required for Inception. |
| NFR-08 | Use pytest and fake-model tests for normal, duplicate, quote-failure, persistence, and failed-save paths. Real-model evaluations are opt-in. Select Hypothesis for focused generated tests of normalization, pure counts, page-reference constraints, and record serialization round trips. These tests complement example-based tests. |
| NFR-09 | Provide readable processing, empty, unavailable-metadata, unverified-quote, offline-model, and error states. Limit interactions to a simple local page with no account system. |
| NFR-10 | Keep .env, data/, temporary outputs, and private papers out of version control. Only commit test PDFs whose redistribution is permitted or synthetic fixtures. |

## Classification examples

- First relevant paper creates a new angle.
- An independent replication belongs to the existing section and adds agreeing evidence.
- A follow-up with a new population adds evidence, potentially extending rather than contradicting earlier work.
- A substantially repeated analysis with no new relevant evidence can be redundant with its earlier source.
- Opposite results from materially different settings are not automatically contradictory.
- An unrelated paper is recorded outside topical sections, with its relevance explanation.

## Success measures

Use a fixed upload order and manual labels established before inspecting model predictions. Two teammates independently label the collection, reconcile disagreements, and record supporting passages. The original target is at least 80 percent primary-label agreement. Report agreement/contradiction/extension quality separately, with counts rather than unsupported generalizations. Verified quotes must match cited pages under the documented normalization. Counts and known year ranges must be exact. Reopening and failed updates must preserve earlier records. No model accuracy or runtime has been measured in this phase.

## Explicit exclusions

No OCR, academic database search, specific-paper recommendations, imbalance commentary, next-reading suggestions, hosted APIs, accounts, shared trackers, multi-section memberships, manual edits, paper removal, question changes, version detection, or complete literature-review generation. Future feature phases in the vision are roadmap possibilities, not commitments in the December 16 MVP.

## Accepted questionnaire choices

Q1 A, Q2 A, Q3 C, Q4 A, Q5 A, Q6 B, Q7 A, Q8 A, Q9 A, Q10 A, Q11 A, Q12 D, Q13 A, Q14 C, Q15 A, Q16 C, Q17 A, Q18 B, Q19 B, Q20 B.

Optional security and resiliency extensions are disabled, while the technical document's security and recovery basics remain requirements. Partial property-based testing is selected. PBT-02, PBT-03, PBT-07, PBT-08, and PBT-09 are carried into Construction. Their implementation checks are not yet applicable during requirements generation. The other PBT rules are advisory under partial mode.

## Remaining Inception work

Requirements require review before User Stories. User Stories, Workflow Planning, Application Design, and Units Generation remain pending. Assignment-rule version provenance and agent-specific installation still require verification. External retrieval of the v1.0.1 source was unavailable in this session, so the existing bundle has not been represented as independently verified.
