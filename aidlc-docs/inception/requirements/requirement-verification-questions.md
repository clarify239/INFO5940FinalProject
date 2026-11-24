# Requirements Clarification Questions: Research Tracker

These questions come from reading `VisionDocument.md`, `TechnicalDocument.md` and `README.md`. Some answer the vision's open questions (Section 8). Others cover details the documents leave unspecified.

Items the documents already decide are **not** asked again. They will go into the requirements as stated:

- The interface is a local web page (FastAPI + Jinja2), not a command-line tool.
- Off-topic papers are saved under "Not relevant or relevance unclear" with a reason, not rejected at upload.
- Redundant papers count toward their assigned section, and the number of redundant papers is also shown separately.
- Manual editing of sections and labels is Phase 2, so MVP users can only inspect them.

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe what you want. Tell me when you are done.

---

## Part A: Classification and sections

## Question 1

Can a paper belong to more than one section? (Vision open question)

A) No. Each paper has exactly one primary section, and counts use only that section

B) Each paper has one primary section and may list secondary "related sections" as links. Counts use only the primary section

C) A paper can fully belong to several sections and is counted in each one

X) Other (please describe after [Answer]: tag below)

[Answer]: B, a paper might contain more than 1 topic, and having a primary section would make it easiest in terms of reasoning about redudancy, but having secondary sections allows for flexibility in papers covering muliple subtopics. 

## Question 2

Where is the line between "Redundant with [paper]" and "Adds to [section]"? (Vision open question)

A) Redundant only when the new paper repeats an existing paper's findings with no new population, method, condition or independent dataset. Any independent replication counts as "Adds to (agrees)"

B) Redundant when the new paper's relevant findings substantially overlap one existing paper, even if its data is independent

C) Let the model decide using a written rubric of examples from our labeled test set, with no hard rule

X) Other (please describe after [Answer]: tag below)

[Answer]: A, because independent data still provides value to the user. Having this yes-or-no answer is also easy to reason about compared to C. 

## Question 3

How much freedom should the agent have when comparing a new paper with the tracker?

A) Real tool-calling agent: Qwen3 calls tools (search passages, read a saved card, list sections) in a loop it controls, with a fixed cap on tool calls per paper

B) Fixed pipeline: code always fetches the most similar cards and passages and gives them to the model in set steps. The model never chooses what to retrieve

C) Hybrid: a fixed pipeline for card writing, then a capped tool-calling loop only for comparison and classification

X) Other (please describe after [Answer]: tag below)

[Answer]: C. Fixed pipeline for cards. Capped tool loop (about 6–8 calls) for comparison and classification, seeded with similar cards, and cited passages must come from tool results.

## Question 4

Section names and summaries are written by the model. When a new paper changes a section, what may be rewritten?

A) The section summary and the overview may be rewritten. The section name stays fixed once created

B) The section name, the section summary and the overview may all be rewritten

C) Only the overview is rewritten. Section summaries are appended to, never rewritten

X) Other (please describe after [Answer]: tag below)

[Answer]: A, since fixed names keep sections easy to navigate and keep links and "Redundant with" labels consistent, while rewriting summaries (instead of appending) lets them work in contradicting findings. The prompt asks for broad section names so they don't end up too narrow.

---

## Part B: Citations and display

## Question 5

How should a quote that fails the verification check appear? (Vision open question)

A) Show it with a clear "Not verified: not found on PDF page N" warning and keep it in the card

B) Hide it from the card and list it only in the inspection view, with the warning

C) Ask the model once to give a replacement quote. If that also fails, show the unverified quote with a warning

X) Other (please describe after [Answer]: tag below)

[Answer]: A and C, since one retry is cheap (it only runs when a quote fails) and usually gives a quote that checks out, so cards end up with fewer warnings. If the retry also fails, the quote is shown with a clear "Not verified" warning along with the pdf name and page number, as the vision requires.

## Question 6

How should citations appear in the tracker and the inspection view? (Vision open question)

A) Inline short tags such as "(Smith 2021, PDF p. 4)" that expand to show the full passage when clicked

B) Numbered footnotes [1], [2] listed under each card or section, each with the paper, PDF page and quoted passage

C) Inline tags in the web page, and numbered footnotes in the exported Markdown

X) Other (please describe after [Answer]: tag below)

[Answer]: C, since inline tags that expand keep the web page readable without leaving the text, while numbered footnotes are the clearest format for a static Markdown file. The export is already its own renderer, so using different formats in each costs little extra.

## Question 7

How should the inspection view let users see source evidence?

A) Show the cited passage text with its PDF page number only

B) Show the cited passage, plus a link that opens the stored PDF at that page in the browser's PDF viewer

C) Show the full extracted text of the cited page with the passage highlighted

X) Other (please describe after [Answer]: tag below)

[Answer]: B, since a link that opens the PDF at the cited page lets users check the quote in its original context, and browsers handle this with no extra work beyond a route that serves stored PDFs (looked up by ID, not file path).

---

## Part C: Trackers, papers and export

## Question 8

The vision says one research question per user, but the planned endpoint uses `tracker_id`. How many trackers does the MVP support?

A) One tracker only. The app opens straight to it, or to a "create tracker" page if none exists

B) Several trackers, each with its own question, chosen from a list on the home page

X) Other (please describe after [Answer]: tag below)

[Answer]: A, since the vision scopes the MVP to one research question per user. We'll keep tracker_id in the data model and endpoints so supporting several trackers later doesn't require a redesign.

## Question 9

Can a user remove a paper from a tracker in the MVP?

A) No. Papers can only be added (removal waits for Phase 2 with the other editing features)

B) Yes. Removing a paper deletes its card, passages and embeddings, and the affected section summary and overview are rewritten

X) Other (please describe after [Answer]: tag below)

[Answer]: A, since removing a paper can break other papers' labels (e.g. papers marked "Redundant with" it, or a section it started), which would need re-classification. Removal waits for Phase 2 with the other editing features.

## Question 10

When is `tracker.md` written?

A) Only when the user clicks "Export"

B) Automatically after every successful paper upload, and also on demand

X) Other (please describe after [Answer]: tag below)

[Answer]: B, since generating the Markdown needs no model call and is instant, so writing it after each successful upload keeps the file always up to date. Export just downloads the current file.

---

## Part D: Processing, failures and the model

## Question 11

Running Qwen3 8B on a laptop may take several minutes per paper. How should the upload work while a paper is being processed?

A) The upload request waits until processing finishes, and the page shows a "Processing…" message

B) The upload returns at once, processing runs in the background, and the page shows step-by-step progress (extracting, writing card, comparing, updating tracker)

X) Other (please describe after [Answer]: tag below)

[Answer]: B, since a request that waits several minutes can time out and gives no feedback, while step-by-step progress shows the app is still working.

## Question 12

What processing time per paper (20 pages or fewer) is acceptable for the MVP on a typical team laptop?

A) Up to 2 minutes

B) Up to 5 minutes

C) Up to 10 minutes

D) No fixed target. Measure it and report it

X) Other (please describe after [Answer]: tag below)

[Answer]: D, since speed depends heavily on each team member's laptop, so a fixed number now would be a guess. We'll time each step on our test papers and report the results, then set a target if needed

## Question 13

If the model returns badly formatted output or processing fails partway through, what should happen?

A) Retry the failing step a fixed number of times (for example 2). If it still fails, save nothing for that paper, leave the tracker exactly as it was, and show an error so the user can try again

B) Same retries, but save whatever finished (for example the card without a classification) and mark the paper "incomplete" so it can be re-run later

X) Other (please describe after [Answer]: tag below)

[Answer]: A, Saving nothing on failure keeps the tracker consistent. Papers can't be removed in the MVP, so a half-saved paper would be stuck in the tracker. This is also simpler than tracking incomplete papers and re-running them.

## Question 14

What is the fallback if Qwen3 8B is too slow or its context is too short on a team laptop? (Vision open question)

A) Make the model name and context length settings in `.env`, so a user can switch to a smaller local model (for example `qwen3:4b`) without code changes

B) Keep Qwen3 8B only and reduce chunk size and batch size until it fits

C) Both: settings in `.env`, plus smaller chunks and batches by default

X) Other (please describe after [Answer]: tag below)

[Answer]: C, Putting the model name and context length in .env lets a slow laptop switch to a smaller model by changing the model in that file. Smaller default chunks and batches also keep each model call short and lower memory use, so the app works on more laptops without too many hardware constraints.

## Question 15

Which model should create the passage embeddings stored in Chroma?

A) A local Ollama embedding model (for example `nomic-embed-text`), pulled once with `ollama pull`, so everything goes through Ollama

B) Chroma's built-in default embedding model (it downloads a small model file the first time it runs)

C) No vector embeddings for the MVP. Use keyword search (SQLite full-text search) for passage retrieval, and drop Chroma

X) Other (please describe after [Answer]: tag below)

[Answer]: A, nomic-embed-text handles longer chunks than Chroma's default model, which cuts off text after about 256 tokens. Users already run Ollama, so pulling one more model keeps setup in one place.

## Question 16

Qwen3 has a "thinking" mode that writes out its reasoning before answering. It is often more accurate but slower. Which should the app use?

A) Thinking off for all steps, to keep processing fast

B) Thinking on only for comparison and classification, and off for card writing and summaries

C) Make it a setting and decide after measuring on our test set

X) Other (please describe after [Answer]: tag below)

[Answer]: C, Thinking mode trades speed for accuracy, and we can't judge that tradeoff without data. Making it a setting in .env lets us test it on our labeled papers and keep whichever setup works best. As a future feature, we can add a toggle on the upload page that lets users choose it themselves depending on their personal laptop constraints.

---

## Part E: Evaluation

## Question 17

The success metrics need a labeled test set of up to 10 papers. Should building an evaluation tool be part of the MVP code?

A) Yes. A script (for example `uv run python -m research_tracker.evaluate`) runs the test papers through the app and reports label agreement, quote verification and count accuracy against the team's manual labels

B) No. The team checks the metrics by hand using the web interface

X) Other (please describe after [Answer]: tag below)

[Answer]: A, We'll re-test the papers each time we change a setting, like thinking mode or chunk size, and checking them by hand every time would be slow. A script makes the results quick to repeat.

---

## Part F: AI-DLC extensions

## Question 18: Security Extensions

Should security extension rules be enforced for this project? (Your technical document already defines security basics. This extension adds a broader set of required checks on top of them.)

A) Yes. Enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)

B) No. Skip all SECURITY rules (suitable for PoCs, prototypes and experimental projects)

X) Other (please describe after [Answer]: tag below)

[Answer]: B, The app runs locally for one user, so production rules like logins and access control don't apply. For a local app the main risks are other people reaching it over the network and data leaving the laptop, and our technical document's security rules already address both.

## Question 19: Property-Based Testing Extension

Should property-based testing (PBT) rules be enforced for this project? (Examples here: quote normalization and verification, chunking with page references, and coverage counts.)

A) Yes. Enforce all PBT rules as blocking constraints (recommended for projects with business logic, data transformations, serialization or stateful components)

B) Partial. Enforce PBT rules only for pure functions and serialization round-trips (suitable for projects with limited algorithmic complexity)

C) No. Skip all PBT rules (suitable for simple CRUD applications, UI-only projects or thin integration layers with no significant business logic)

X) Other (please describe after [Answer]: tag below)

[Answer]: B, Quote verification, chunking, and coverage counts are implemented with pure functions where property tests are helpful for catching edge cases. The rest of the app depends on the model's output, which our fake-model unit tests and evaluation script will cover instead. 

## Question 20: Resiliency Extensions

Should the resiliency baseline be applied to this project?

**What this extension is.** It applies a set of **directional, design-time best practices** for building resilient systems, based on the **AWS Well-Architected Framework (Reliability Pillar)** and resilience-review guidance. It steers requirements, design and code toward fault tolerance, high availability, observability and recoverability, across 15 practice areas.

**What this extension is NOT.** It does **not** make your workload production-ready, and it does not guarantee any availability, RTO or RPO target. Note that much of it targets cloud workloads, while this app runs only on a laptop.

A) Yes. Apply the resiliency baseline as directional best practices and design-time guidance (recommended for business-critical workloads)

B) No. Skip the resiliency baseline (suitable for PoCs, prototypes and experimental projects where rapid iteration matters more than reliability)

X) Other (please describe after [Answer]: tag below)

[Answer]: B, The resiliency baseline is mostly aimed at cloud systems/deployed applications, while our app runs locally. The failures that matter for us, like bad model output and running out of memory, are already handled by our answers to Q11 and Q13.
