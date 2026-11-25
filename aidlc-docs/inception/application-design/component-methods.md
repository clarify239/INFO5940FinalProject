# Component Methods: Research Tracker

Signatures are written in Python 3.12 style. Types refer to `models.py` (see components.md). These signatures are the contracts between units. Business rules, such as what makes a classification valid or exactly how text is normalized, are defined in each unit's Functional Design. Names may be refined there, but changing a signature that crosses units needs the other unit's owner to agree.

## 1. Settings

```python
class Settings(BaseSettings):          # pydantic-settings, reads .env
    model_name: str                    # e.g. "research-tracker-qwen3" (built from the Modelfile)
    embedding_model: str               # "nomic-embed-text"
    context_length: int                # must match the Modelfile (A-4)
    thinking: bool
    data_dir: Path                     # default "data"
    ollama_base_url: str               # default "http://localhost:11434/v1"; must be localhost
    tuning: Tuning                     # chunk size, batch size, model-call cap, min quote words, min text chars, seed count

def load_settings(overrides: dict | None = None) -> Settings
    """Load and validate settings; raise SettingsError naming the bad setting. Evaluation passes data_dir override."""
```

## 2. PDF Intake

```python
class PdfRejected(Exception):
    message: str                       # plain-language, shown on the upload form

def check_upload(data: bytes, original_filename: str, store: TrackerStore,
                 tracker_id: int, settings: Settings) -> ExtractedPdf
    """Run every FR-2.2 check and the duplicate check during the request; raise PdfRejected."""

def extract_pages(data: bytes, settings: Settings) -> list[str]
    """Open with pypdf, reject encrypted or too-long PDFs, return page texts (index 0 = PDF page 1)."""

def sha256_of(data: bytes) -> str
```

## 3. Indexer

```python
def chunk_pages(pages: list[str], settings: Settings) -> list[Chunk]
    """Split pages into chunks that keep their 1-based PDF page numbers. Pure."""

def embed_chunks(chunks: list[Chunk], client: ModelClient) -> list[list[float]]
def embed_text(text: str, client: ModelClient) -> list[float]

class InMemoryPassages:
    def __init__(self, chunks: list[Chunk], embeddings: list[list[float]]) -> None
    def search(self, query_embedding: list[float], k: int) -> list[Passage]
        """Search the new paper's own chunks (used by search_passages with scope="new")."""
```

## 4. Model Client

```python
class ModelError(Exception): ...       # connection failure, timeout, HTTP error

class ModelReply(BaseModel):
    content: str | None
    tool_calls: list[ToolCall]         # name + JSON arguments; empty if none

class ModelClient(Protocol):
    def chat(self, step: str, messages: list[Message], *,
             tools: list[ToolSpec] | None = None,
             schema: type[BaseModel] | None = None) -> ModelReply
        """One model call. `step` names the pipeline step (used by the fake and by timing logs)."""
    def embed(self, texts: list[str]) -> list[list[float]]

class OllamaClient(ModelClient):
    def __init__(self, settings: Settings) -> None
        """openai.OpenAI(base_url=settings.ollama_base_url); applies thinking mode and schema as confirmed in Functional Design."""

class FakeModelClient(ModelClient):
    def __init__(self, script: dict[str, list[ModelReply | Exception]],
                 embedding: Callable[[str], list[float]] | None = None) -> None
        """Returns the next scripted reply for each step, in order; raises ScriptExhausted when a step's list runs out."""
    calls: list[tuple[str, list[Message]]]   # recorded for assertions
```

## 5. Card Writer

```python
def write_card(pages: list[str], chunks: list[Chunk], question: str,
               client: ModelClient, settings: Settings) -> Card
    """Fixed sequence of model calls -> validated Card; every quote passes through check_with_replacement.
    Raises StepFailed on invalid output (retried by the Processor)."""
```

## 6. Quote Checker

```python
def normalize(text: str) -> str
    """Join line-break hyphenation, collapse whitespace. Pure."""

def check_quote(quote: str, pages: list[str], pdf_page: int, min_words: int) -> QuoteStatus
    """verified | not_found | too_short. Pure."""

def check_with_replacement(quote: Quote, pages: list[str], context: str,
                           client: ModelClient, settings: Settings) -> Quote
    """If the quote fails, ask the model once for a replacement and check it the same way (FR-5.2)."""
```

## 7. Comparison Agent

```python
class ComparisonFailed(Exception): ...

def classify(card: Card, new_passages: InMemoryPassages, card_embedding: list[float],
             question: str, store: TrackerStore, client: ModelClient,
             settings: Settings) -> Classification
    """FR-6.0 single call if there are no sections; otherwise the capped loop, then the final tool-free call."""

def classify_first(card: Card, new_passages: InMemoryPassages, question: str,
                   client: ModelClient) -> Classification
    """New angle or Not relevant, citing the new paper only (FR-6.0)."""

def run_loop(card: Card, seeds: list[PaperRecord], tools: ToolBox, question: str,
             client: ModelClient, settings: Settings) -> Classification | None
    """Model calls up to the cap; a failed turn is re-asked up to 2 times, each counting toward the cap.
    Returns None if the loop ends without a valid label."""

def final_call(card: Card, evidence: list[Passage], question: str,
               client: ModelClient) -> Classification
    """Tool-free call; retried up to 2 more times with the same evidence (FR-6.4)."""

def validate_classification(raw: ClassificationOutput, seen: set[PassageRef],
                            store: TrackerStore) -> Classification
    """Label kind, existing relevant targets, retrieved-only citations, both-paper rule. Raises InvalidOutput."""
```

## 8. Agent Tools

The model sees these three tools (names, arguments and results are part of the contract between the agent and storage units):

| Tool | Arguments | Returns |
|---|---|---|
| `search_passages` | `query: str`, `scope: "saved" \| "new"` (default `"saved"`), `k: int` (capped in code) | list of `{paper_id, display_name, chunk_id, pdf_page, text}` |
| `read_card` | `paper_id: int` | `{paper_id, display_name, section, title, authors, year, findings, methods, limitations}` |
| `list_sections` | none | list of `{section_id, name, summary, paper_count}` |

```python
class ToolBox:
    def __init__(self, store: TrackerStore, new_passages: InMemoryPassages,
                 client: ModelClient, settings: Settings) -> None
    specs: list[ToolSpec]                         # JSON schemas passed to the model (Q7)
    seen_passages: set[PassageRef]                # every passage returned, for citation checks (FR-6.5)
    def call(self, name: str, arguments: dict) -> dict
        """Run one tool; unknown tool or bad arguments raise InvalidToolCall. Read-only.
        Saved-scope results come only from committed papers that have a primary section."""
```

## 9. Tracker Updater

```python
def plan_update(classification: Classification, card: Card, question: str,
                store: TrackerStore, client: ModelClient, settings: Settings) -> TrackerUpdate
    """Pick the new or existing section, rewrite its summary (in batches) and the overview; never rename a section."""
```

## 10. Coverage

```python
def compute_coverage(papers: list[PaperRecord], sections: list[Section]) -> Coverage
    """Primary-section counts, redundant count, total, year range, unknown-year count. Pure."""
```

## 11. Markdown Exporter

```python
def render_markdown(view: TrackerView) -> str
    """tracker.md text with numbered footnotes; unverified quotes marked. Pure, no model call."""
```

## 12. Storage

```python
class SchemaMismatch(Exception): ...           # "Data folder uses an older format; delete data/ to start over."

class TrackerStore:
    def __init__(self, settings: Settings) -> None
        """Opens SQLite (per-thread connections), Chroma (telemetry off) and the file folder under settings.data_dir."""

    # Tracker
    def create_tracker(self, question: str) -> int
    def get_tracker(self) -> Tracker | None                     # one tracker in the MVP
    def get_tracker_by_id(self, tracker_id: int) -> Tracker | None

    # Committed reads (only committed papers are ever returned)
    def paper_count(self, tracker_id: int) -> int
    def paper_exists(self, tracker_id: int, sha256: str) -> bool
    def list_sections(self, tracker_id: int) -> list[Section]
    def list_papers(self, tracker_id: int) -> list[PaperRecord]
    def get_paper(self, paper_id: int) -> PaperRecord | None
    def get_pages(self, paper_id: int) -> list[str]             # stored page text, for checks (FR-17.7)
    def relevant_paper_ids(self, tracker_id: int) -> set[int]   # committed and has a primary section
    def similar_cards(self, embedding: list[float], allowed: set[int], k: int) -> list[PaperRecord]
    def search_passages(self, embedding: list[float], allowed: set[int], k: int) -> list[Passage]
    def tracker_view(self, tracker_id: int) -> TrackerView
    def pdf_path(self, paper_id: int) -> Path | None            # by ID only (FR-12.4)

    # All-or-nothing save (Q4)
    def commit(self, tracker_id: int, draft: PaperDraft) -> PaperRecord
        """Write PDF file and Chroma entries, then one SQLite transaction that saves all rows and marks the paper committed."""
    def write_tracker_md(self, tracker_id: int) -> Path          # render_markdown(tracker_view) -> file
    def cleanup_uncommitted(self) -> list[Notice]
        """Start-up: delete files and Chroma entries with no committed paper; turn leftover job records into notices."""

    # Processing-job record and notices (saved outside the paper's commit, FR-11.6)
    def start_job(self, tracker_id: int, sha256: str, original_filename: str) -> int
    def finish_job(self, job_id: int) -> None                    # on success or failure
    def list_notices(self, tracker_id: int) -> list[Notice]
    def dismiss_notice(self, notice_id: int) -> None             # deletes the record entirely

    # Timing log
    def append_timing(self, record: TimingRecord) -> None
```

## 13. Paper Processor (service)

```python
class ProcessingFailed(Exception):
    step: str
    message: str                       # plain-language, shown on the page

ProgressCallback = Callable[[str], None]   # receives "indexing" | "writing_card" | "comparing" | "updating_tracker"

class PaperProcessor:
    def __init__(self, store: TrackerStore, client: ModelClient, settings: Settings) -> None
    def process(self, tracker_id: int, extracted: ExtractedPdf,
                on_progress: ProgressCallback = lambda step: None) -> PaperRecord
        """Run all steps with retries, commit, write tracker.md; raise ProcessingFailed with nothing saved."""
```

## 14. Job Runner

```python
class Busy(Exception): ...             # "Wait until the current paper finishes."

class JobRunner:
    def __init__(self, processor: PaperProcessor, store: TrackerStore) -> None
    def start(self) -> None                         # starts the worker thread
    def stop(self, timeout: float) -> None          # on app shutdown
    def submit(self, tracker_id: int, extracted: ExtractedPdf) -> None
        """Raise Busy if a paper is processing; otherwise save the job record and hand it to the worker."""
    def status(self) -> JobStatus                   # read by the polling endpoint
    def is_busy(self) -> bool
```

## 15. Web (HTTP routes)

| Method and path | Purpose | Success | Errors |
|---|---|---|---|
| `GET /` | Create-tracker page if none exists, otherwise the tracker page | 200 HTML | — |
| `POST /trackers` | Create the tracker from the form's question | 303 to `/` | 400 empty question; 409 tracker exists |
| `POST /api/trackers/{tracker_id}/papers` | Upload one PDF; checks run in the request | 202 with `JobStatus` | 400, 413, 422 `PdfRejected`; 409 duplicate; 409 busy; 404 tracker |
| `GET /api/trackers/{tracker_id}/status` | Current `JobStatus` for polling | 200 JSON | 404 tracker |
| `GET /papers/{paper_id}/pdf` | Serve the stored PDF by ID (link adds `#page=N`) | 200 PDF | 404 unknown ID |
| `GET /trackers/{tracker_id}/export` | Download `tracker.md` | 200 Markdown attachment | 404 tracker |
| `POST /notices/{notice_id}/dismiss` | Dismiss an interrupted-paper notice | 303 to `/` | 404 unknown ID |

The technical document illustrates the same HTTP 202 response. Clients poll the status endpoint and read the finished card on the tracker page.

```python
def create_app(settings: Settings | None = None, client: ModelClient | None = None) -> FastAPI
    """Start-up order: settings -> schema check -> cleanup_uncommitted -> write_tracker_md -> JobRunner.start.
    Tests pass a FakeModelClient and a temporary data_dir."""
```

## 16. Evaluation

```python
def main(argv: list[str]) -> int
    """`run [--labels PATH] [--papers DIR]` or `score-checklist [PATH]`."""

def run(labels_path: Path, papers_dir: Path, settings: Settings, client: ModelClient) -> EvalReport
    """Clear eval-data/, create the tracker, process papers in label-file order via PaperProcessor, then build the report."""

def label_agreement(labels: list[ExpectedLabel], papers: list[PaperRecord]) -> LabelResult      # SC-3
def count_check(store: TrackerStore, tracker_id: int) -> CountResult                            # SC-5
def write_checklist(store: TrackerStore, tracker_id: int, path: Path) -> None                   # FR-17.6
def score_checklist(path: Path) -> ChecklistResult                                               # SC-2
def stored_quote_check(store: TrackerStore, tracker_id: int, tracker_md: Path) -> list[QuoteMismatch]  # FR-17.7
def timing_summary(timing_log: Path) -> TimingSummary                                            # FR-16, US-28
def render_report(report: EvalReport) -> str                                                     # includes settings used (US-29)
```
