# Requirements Clarification Questions: Research Tracker

All 20 answers are complete and none of them contradict each other. A few answers raise follow-up details. Some are gaps between two answers, and one is a technical limit of the planned setup. These need a decision before I write `requirements.md`.

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe what you want.

---

## Ambiguity 1: When secondary sections are assigned (Q1)

You chose one primary section plus secondary "related sections" (Q1: B). Q4 says section names are fixed, and the vision says earlier cards are preserved. It is not yet clear whether a new section can also be linked back to papers that were uploaded earlier.

### Clarification Question 1

When a later paper starts a new section, can earlier papers gain it as a secondary section?

A) No. Secondary sections are assigned only when a paper is uploaded, and earlier papers never change. Simple, and fully consistent with "earlier cards are preserved"

B) Yes. When a new section is created, the model may add it as a secondary link on earlier papers. Their cards, primary sections and labels stay unchanged; only the list of links grows

X) Other (please describe after [Answer]: tag below)

[Answer]:

---

## Ambiguity 2: A card whose quotes still fail after the retry (Q5 vs Q13)

Q5 says a quote that fails after one retry is shown with a "Not verified" warning. Q13 says a failed step saves nothing. It is unclear which rule wins when every quote on a card fails verification.

### Clarification Question 2

If no quote on a card passes verification, even after the retry, what happens?

A) The card is still saved, with every quote marked "Not verified". Failed quotes are a warning, not a processing failure

B) Treat it as a processing failure under Q13: save nothing and show an error

C) Save the card only if at least one quote is verified. Otherwise treat it as a failure

X) Other (please describe after [Answer]: tag below)

[Answer]:

---

## Ambiguity 3: The "PDF name" in the warning (Q5)

Q5 says the warning shows "the pdf name". The technical document stores each PDF under its SHA-256 hash, not its filename.

### Clarification Question 3

What name should be shown for a paper in warnings and citations?

A) The original upload filename, saved in the database for display only. The file on disk is still named by its hash

B) The paper's extracted title, falling back to the original filename if the title is "unavailable"

X) Other (please describe after [Answer]: tag below)

[Answer]:

---

## Ambiguity 4: The tool loop reaches its cap without a label (Q3 vs Q13)

Q3 caps the comparison loop at about 6–8 tool calls. Q13 retries a failing step and then saves nothing. It is unclear what happens if the model uses every call and never gives a final classification.

### Clarification Question 4

What happens when the tool-call cap is reached without a valid final classification?

A) Make one last model call with tools turned off, asking for the classification from the evidence already gathered. If that output is still invalid, it counts as a failed step under Q13 (retry, then save nothing)

B) Treat reaching the cap as a failed step right away, and retry the whole comparison loop under Q13

X) Other (please describe after [Answer]: tag below)

[Answer]:

---

## Ambiguity 5: Uploading again while a paper is processing (Q11)

Background processing (Q11: B) lets the user use the page while a paper is being processed. The vision says papers are uploaded one at a time. Under Q13, a paper interrupted by an app shutdown saves nothing.

### Clarification Question 5

What happens if the user tries to upload another paper while one is still processing?

A) Uploading is blocked, with a message like "Wait until the current paper finishes," until processing ends

B) The new upload waits in a queue and is processed after the current one, one at a time

X) Other (please describe after [Answer]: tag below)

[Answer]:

---

## Ambiguity 6: Evaluation data and timing (Q8, Q12, Q17)

There is one tracker (Q8: A), and the evaluation script runs the test papers through the app (Q17: A). If the script uses the real data folder, it could mix test papers into the user's tracker. Q12 also says step times will be measured and reported, but not by what.

### Clarification Question 6

Where should the evaluation script store its data?

A) In its own temporary data folder, never touching the user's tracker in `data/`

B) In a named folder the team chooses (for example `eval-data/`), kept after the run so results can be inspected in the web page. It is also listed in `.gitignore`

X) Other (please describe after [Answer]: tag below)

[Answer]:

### Clarification Question 7

How should step times (Q12) be measured?

A) The app logs how long each step takes on every upload (extract, card, compare, update), and the evaluation script prints a timing summary from those logs

B) Only the evaluation script measures and reports timing. Normal uploads do not log times

X) Other (please describe after [Answer]: tag below)

[Answer]:

---

## Technical issue: Setting the context length (Q14)

Q14 puts the model's context length in `.env`. According to Ollama's documentation, its OpenAI-compatible API (which the `openai` library uses) does not accept a context-size setting per request. The context size is set on the Ollama side instead. The team should confirm this against the current Ollama docs during design, but it affects the requirements, so the approach needs choosing now.

### Clarification Question 8

How should the context length be applied?

A) The app uses the `.env` value only to size its chunks and batches. Setup instructions tell users to start Ollama with the matching `OLLAMA_CONTEXT_LENGTH` environment variable

B) The repository includes an Ollama Modelfile that creates a custom model (for example `research-tracker-qwen3`) with the context size built in. `.env` names that model, and setup adds one `ollama create` command

C) Call Ollama's native API, which accepts context size per request. This needs an HTTP client alongside or instead of the `openai` library, which changes the technical document

X) Other (please describe after [Answer]: tag below)

[Answer]:
