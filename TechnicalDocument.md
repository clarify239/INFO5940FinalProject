# Technical Environment: Research Tracker

## 1. Language, package manager, and framework

All application code is written in Python 3.12. Our package manager is uv, and the application is served as a local web app built on FastAPI (served by Uvicorn).

- **Language:** Python 3.12. Can install with uv (`uv python install 3.12`)
- **Package manager:** Dependencies will be declared in `pyproject.toml` and pinned in `uv.lock`, and both files will be committed to the GitHub repo. Setup is `uv sync`, commands run through `uv run`
- **Framework:** One process serves the API endpoints and the local web page. The page uses Jinja2, which lets us write the HTML layout for a paper card once and reuse it for every paper in the tracker, inserting each paper's own title, authors and quotes

We call the LLM model with the `openai` Python library, pointed at Ollama, and read PDF text with `pypdf`.

## 2. Cloud provider and deployment model

There is no cloud provider: the app is deployed locally on each user's laptop, and no request leaves the machine.

- **Research Tracker web app (FastAPI + Uvicorn):** runs on the user's laptop at `127.0.0.1:8000` (Uvicorn's default port)
- **Qwen3 8B via Ollama:** runs on the user's laptop, reached at `http://localhost:11434/v1` (OpenAI-compatible API)
- **Saved data:** the SQLite database (paper cards, sections, quotes), the Chroma vector database (passage embeddings), uploaded PDFs and exported `tracker.md` live on local disk in the project's `data/` folder.

To install, a user clones the repository, run: `ollama pull qwen3:8b` and `uv sync`, then starts the app with `uv run uvicorn research_tracker.main:app`. There is no hosting, no accounts and no build pipeline.

## 3. Test framework

Tests use pytest, run with `uv run pytest`, and must pass before a pull request is merged.

We have two kinds of tests:

- **Unit tests** use a fake model (a stand in for Qwen3 that returns answers we wrote in advance) so they are fast and give the same result every time
- **Model tests** use the real Qwen3 model to check it actually works with our prompts. They are slow, so they only run when you ask

To test endpoints (like the PDF upload), unit tests use FastAPI's `TestClient`. It sends a request to the app the way a browser would, without starting the server, and checks the response.

## 4. Prohibited libraries

The libraries below may not be added to `pyproject.toml`:

- **LangChain, LlamaIndex**
  - *Reason:* they hide the prompts and tool-call loop behind layers of abstraction, which makes any unexpected model outputs hard to debug. They also install many extra libraries and change often so existing code would break
  - *Use instead:* Ollama's OpenAI-compatible API to send prompts to Qwen3, plus a few Python functions we write ourselves to handle each step (summarizing articles, comparing articles etc.)
- **Hosted LLM APIs (anthropic, google-genai etc.)**
  - *Reason:* Instead of handling requests locally, the model would be hosted on a third-party server.
  - *Use instead:* We plan to host the LLM model locally with Ollama
- **PyPDF2**
  - *Reason:* This library is deprecated. It was merged into pypdf
  - *Use instead:* pypdf
- **PyMuPDF**
  - *Reason:* Its AGPL license would require releasing the whole project under the AGPL
  - *Use instead:* pypdf
- **Pytesseract and other OCR Libraries**
  - *Reason:* scanned PDFs are out of scope
  - *Use instead:* reject PDFs with little to no extractable text, with a clear error
- **Pickle, shelve for tracker**
  - *Reason:* the app stores each user's paper notes in a file on their laptop so they are still there next time. If the file was tampered, pickle would run any harmful code hidden in it when the app opens it. The Python documentation itself warns against this.
  - *Use instead:* SQLite database to store information about evidence, major topics etc. When new information is added, SQLite saves only the new part, while pickle rewrites the whole file. So if the app crashes mid-save, SQLite keeps all earlier work, but pickle can lose it.

## 5. Security Basics

The main risks are an exposed local server and hostile PDF uploads. The following lists ways we will try to reduce risks:

- **Network Exposure:** Uvicorn binds to `127.0.0.1` (reachable only from whichever laptop the app is running on), Ollama keeps its default localhost setting (never `0.0.0.0` which listens to all network interfaces), and Chroma's anonymous usage reporting, which sends stats to Chroma's company over the internet, is off (`anonymized_telemetry=False`). This way the app and model only accept connections from the user's own laptop, so others on the same Wi-Fi can't open the app or use the model.
- **Uploads:** `%PDF-` is the marker at the very start of every real PDF file's contents. Checking this ensures the application only accepts real PDFs. We will reject encrypted PDFs and any PDF over 20 pages. PDFs are saved under its SHA-256 hash instead of the user's filename to prevent path manipulation
- **Prompt Injection:** A paper could contain hidden text and malicious instructions. So paper text is always given to Qwen3 as material to read and never as instructions. Qwen3 can only look up papers already uploaded in the app; it cannot touch files, execute commands, or use the internet
- **Rendering:** a paper (or Qwen3's output) could contain web code such as `<script>`. If the page displayed it as-is, the browser would run that code inside our app. Jinja2's autoescaping stays on so such text is shown as plain text, never run.
- **Database:** to prevent SQL injection (text being saved gets run as a database command), all text is passed to SQLite through `?` placeholders, so it is always stored as plain data.
- **Secrets in git:** Any API keys, secrets etc. are placed in `.env`. `.env` and `data/` are listed under `.gitignore` so secrets and papers are never committed

## 6. Examples

### Typical endpoint

`POST /api/trackers/{tracker_id}/papers` accepts one PDF as multipart form data and returns the new paper's card and label. This is planned code: the helpers it imports (`get_db`, `extract_pages`, `process_paper`) will be built in their own modules.

```python
import hashlib

from fastapi import APIRouter, Depends, HTTPException, UploadFile

from research_tracker.db import Database, get_db
from research_tracker.models import PaperResult
from research_tracker.pdf import PdfRejected, extract_pages
from research_tracker.pipeline import process_paper

MAX_UPLOAD_BYTES = 25 * 1024 * 1024
router = APIRouter()


@router.post("/api/trackers/{tracker_id}/papers", status_code=201, response_model=PaperResult)
def upload_paper(tracker_id: int, file: UploadFile, db: Database = Depends(get_db)) -> PaperResult:
    tracker = db.get_tracker(tracker_id)
    if tracker is None:
        raise HTTPException(404, "Tracker not found")

    data = file.file.read(MAX_UPLOAD_BYTES + 1)
    if len(data) > MAX_UPLOAD_BYTES or not data.startswith(b"%PDF-"):
        raise HTTPException(400, "Upload a PDF file under 25 MB")

    sha256 = hashlib.sha256(data).hexdigest()
    if db.paper_exists(tracker_id, sha256):
        raise HTTPException(409, "This PDF is already in the tracker")

    try:
        pages = extract_pages(data)  # list of page texts; index 0 is PDF page 1
    except PdfRejected as err:  # encrypted, over 20 pages, or too little text
        raise HTTPException(422, str(err))

    return process_paper(db, tracker, pages, sha256)
```

### Typical function

`verify_quote` checks that a quote the model gives really appears on the PDF page it cites.

```python
import re

_HYPHEN_BREAK = re.compile(r"(\w)-\s*\n\s*(\w)")
_WHITESPACE = re.compile(r"\s+")


def normalize(text: str) -> str:
    """Join words split by line-break hyphenation, then collapse whitespace."""
    text = _HYPHEN_BREAK.sub(r"\1\2", text)
    return _WHITESPACE.sub(" ", text).strip()


def verify_quote(quote: str, pages: list[str], pdf_page: int) -> bool:
    """Return True if the quote appears on the 1-based PDF page."""
    if not 1 <= pdf_page <= len(pages):
        return False
    needle = normalize(quote)
    return bool(needle) and needle in normalize(pages[pdf_page - 1])
```

### Typical test

Tests are small, named for the behavior they check, and use inline sample text instead of real PDFs or the model.

```python
import pytest

from research_tracker.quotes import verify_quote

PAGES = [
    "Abstract\nWe surveyed 1,200 adolescents aged 13 to 17.",
    "Heavy social media use was associ-\nated with higher depressive\nsymptoms in girls.",
]


def test_quote_matches_across_hyphenated_line_break():
    assert verify_quote("associated with higher depressive symptoms", PAGES, pdf_page=2)


def test_quote_on_wrong_page_is_not_verified():
    assert not verify_quote("associated with higher depressive symptoms", PAGES, pdf_page=1)


@pytest.mark.parametrize("pdf_page", [0, 3])
def test_out_of_range_page_is_not_verified(pdf_page):
    assert not verify_quote("We surveyed", PAGES, pdf_page=pdf_page)
```