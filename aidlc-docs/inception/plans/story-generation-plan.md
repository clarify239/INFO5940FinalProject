# Story Generation Plan: Research Tracker

This plan describes how the requirements in `aidlc-docs/inception/requirements/requirements.md` will be turned into user stories and personas. Answer the questions in Part B first. Once you approve the plan, I'll carry out the checklist in Part C.

---

## Part A: Breakdown Approaches

| Approach | How stories are grouped | Strengths | Weaknesses for this project |
|---|---|---|---|
| **User journey** | Follows the core journey: set up → create tracker → upload → read results → inspect evidence → export → return later | Matches the SC-1 demo; easy to read end to end | One step (upload and processing) holds most of the logic and becomes very large |
| **Feature** | One group per system capability (upload, cards, quotes, classification, counts, export…) | Maps closely to the FR sections | Hides how features connect from the user's side; can drift into technical tasks |
| **Persona** | Groups by user type | Useful when user types need different things | The MVP has essentially one end user, so groups would be lopsided |
| **Domain** | Groups by business area (papers, evidence, tracker, evaluation) | Clean boundaries | Close to the feature approach; adds little here |
| **Epic** | Epics, each holding several smaller stories | Scales well; epics can line up with the five ownership areas in the README | Needs a rule for which epic a cross-area story belongs to |

A hybrid is possible, for example epics ordered along the user journey.

---

## Part B: Planning Questions

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe what you want.

## Question 1

Which personas should the stories be written for?

A) One end-user persona: a student researcher writing a literature review

B) Two end-user personas, an undergraduate new to research and a graduate student or research assistant with more experience, plus no developer persona

C) The end-user personas from B, plus a "team evaluator" persona for the evaluation script, setup and test-checklist stories

D) One end-user persona plus the "team evaluator" persona

X) Other (please describe after [Answer]: tag below)

[Answer]: D. Two end-user personas wouldn't need different things in the MVP: same limits, same screens, same journey. Each story would be written twice with almost no difference. A "team evaluator" persona is still useful, because the evaluation script, step timing (FR-16/17) and the quote checklist don't fit a student's goals. Setup stories should stay with the student, since they're the one installing the app.

## Question 2

How technically comfortable should the end-user persona be assumed to be? This shapes the setup stories and how error messages are written.

A) Comfortable with a terminal: can clone a repository and run `ollama` and `uv` commands by following a README

B) Not technical: needs step-by-step setup instructions and plain-language error messages with no jargon

X) Other (please describe after [Answer]: tag below)

[Answer]: A. Setup has no installer: it means cloning the repository and running ollama pull, ollama create and uv sync. Assuming a non-technical user would call for an installer that the requirements don't include. Error messages still have to be plain language under NFR-6.1 either way, so A loses nothing there.

## Question 3

How should the stories be organized?

A) User journey only (see Part A)

B) Feature-based only

C) Epics ordered along the user journey, for example: Set up and start → Create tracker → Upload a paper → Read the card and label → Inspect evidence → See coverage → Export and return later → Evaluate the app

D) Epics that match the five ownership areas in the README (PDF processing and retrieval, paper cards, comparison and classification, summaries and coverage, interface and persistence)

X) Other (please describe after [Answer]: tag below)

[Answer]: C. Splitting into epics keeps the upload step from becoming one huge story. D follows how the team divides the work, which is about who builds what, not what the user experiences. That split should belong in the later step that breaks the system into units. 

## Question 4

How big should each story be?

A) Coarse: about 8–12 stories, roughly one per epic or major capability

B) Medium: about 20–30 stories, one per user-visible behavior (for example "see why a quote is not verified" is its own story)

C) Fine: 40 or more stories, including one per error case

X) Other (please describe after [Answer]: tag below)

[Answer]: B. About 20–30 stories fits the 17 requirement groups. Coarse stories would hide behaviors that need their own checks, like an unverified quote. 40+ stories is too many to keep track of with four people and a December 16 deadline.

## Question 5

How should failure and edge cases be handled? Examples: rejected uploads, an off-topic first paper, unverified quotes, failed processing, and a restart during processing.

A) As acceptance criteria inside the related main story

B) As separate stories when the user sees a distinct result (for example "upload is rejected" or "processing failed, try again"), and as acceptance criteria otherwise

C) Always as separate stories

X) Other (please describe after [Answer]: tag below)

[Answer]: B. A rejected upload and "processing failed, try again" each show the user a different screen, so they should get their own stories. Smaller cases fit better as acceptance criteria on the main story with one unverified quote, cleanup after a restart, and an off-topic first paper. This also keeps the count near Q4's range.

## Question 6

What format should the acceptance criteria use?

A) Given / When / Then scenarios

B) A checklist of testable statements

C) Given / When / Then for behavior that involves the model or several steps; a checklist for simple rules (for example limits and counts)

X) Other (please describe after [Answer]: tag below)

[Answer]: C. Given/When/Then makes multi-step and model behavior clear. Writing it out for simple limits like "reject PDFs over 20 pages" just adds length, and a checklist line says the same thing.

## Question 7

Should acceptance criteria name the model's exact output, given that model results vary?

A) No. Criteria about model output check structure and rules only, never content. For example, "the label is one of the four allowed labels and has a one-sentence reason citing passages from both papers," not "the label is Adds to." Correct content is measured by the evaluation script (SC-3)

B) Yes. Use concrete expected labels for specific test papers

X) Other (please describe after [Answer]: tag below)

[Answer]: A. Model output varies between runs, and the unit tests use a fake model (NFR-4.2), so they can only check structure and rules. Whether the labels are correct is what SC-3 and the evaluation script measure. Naming exact labels in stories would make them fail at random.

## Question 8

How should stories be traced back to the requirements?

A) Each story lists the FR, NFR and SC IDs it covers, and a table at the end of `stories.md` shows that every FR is covered by at least one story

B) Each story lists its IDs, but no coverage table

C) No ID references

X) Other (please describe after [Answer]: tag below)

[Answer]: A. We've traced every requirement back to its source so far, and a coverage table is a cheap way to catch an FR that no story covers. Maybe we could extend the table to cover NFRs and success criteria too, not just FRs. Step 5.3 currently checks "every FR" only.

## Question 9

Requirements such as localhost-only binding, autoescaping and SQL placeholders have no user-visible behavior. How should they appear?

A) As acceptance criteria on the story they protect (for example, autoescaping on the "read the card" story)

B) In a separate "constraints that apply to every story" list at the top of `stories.md`, and not repeated in each story

C) As their own stories written for the team evaluator persona

X) Other (please describe after [Answer]: tag below)

[Answer]: B. Localhost binding, autoescaping and SQL placeholders apply to every story. Repeating them in 25 stories is not necessary. Rules that belong to one feature are already FRs, like serving PDFs only by ID (FR-12.4) and the upload checks (FR-2.2), so they still show up as acceptance criteria on their own stories.

## Question 10

Should each story carry a priority or demo tag?

A) Yes. Mark the stories needed for the SC-1 end-to-end demo as "Demo path," so the team can make sure those work first

B) Yes. Use MoSCoW (Must / Should / Could) within the MVP

C) No. Everything in the requirements is MVP scope, so there are no tags

X) Other (please describe after [Answer]: tag below)

[Answer]: A. Everything in the requirements is MVP scope, so MoSCoW labels would be arbitrary. A "Demo path" tag answers the useful question: what has to work first for SC-1.

---

## Part B2: Decisions Applied

These come from the Part B answers and `story-plan-clarification-questions.md`.

- **Personas**: one end-user persona (student researcher, comfortable with a terminal) and one "team evaluator" persona. Setup stories belong to the student (Q1, Q2)
- **Organization**: epics ordered along the user journey. Set up and start → Create tracker → Upload a paper → Read the card and label → Inspect evidence → See coverage → Export and return later → Evaluate the app (Q3)
- **Size**: about 20–30 stories, one per user-visible behavior (Q4)
- **Failure and edge cases**: separate stories when the user sees a distinct result (upload rejected; processing failed, try again). Otherwise they are acceptance criteria on the main story, for example one unverified quote or an off-topic first paper. The restart notice (FR-11.6) is an acceptance criterion on the "processing failed, try again" story (Q5, CQ2)
- **Acceptance criteria**: Given / When / Then for behavior that involves the model or several steps; a checklist for simple rules (Q6)
- **Model output**: criteria check structure and rules only, never specific labels or wording (Q7)
- **Traceability**: each story lists its FR, NFR and SC IDs. A coverage table covers every FR, NFR and SC. Each row points to a story, the shared constraints list, or "Team practice: checked in Build and Test" (Q8, CQ1)
- **Shared constraints**: runtime rules that apply to every story (localhost binding, no outside requests, autoescaping, SQL placeholders, paper text never treated as instructions) are listed once at the top of `stories.md` (Q9)
- **Tags**: stories needed for the SC-1 end-to-end demo are tagged "Demo path" (Q10)
- **Dependencies**: each story has a "Depends on" field that lists only hard dependencies by story ID, meaning another story whose behavior must exist before this one can be seen working. Soft links ("related to", "same screen") are left out. A story is still testable on its own with seeded data or the fake model (follow-up discussion)

---

## Part C: Execution Checklist

This will run after the plan is approved. Steps that depend on your answers are worded generally for now.

- [x] **1. Prepare**
  - [x] 1.1 Re-read `requirements.md` and list every FR, NFR and SC ID
  - [x] 1.2 Apply the answers from Part B (personas, organization, size, format)
- [x] **2. Personas** (`aidlc-docs/inception/user-stories/personas.md`)
  - [x] 2.1 Write each persona: goals, context, technical comfort, frustrations, what success looks like
  - [x] 2.2 State the MVP limits each persona works within (one tracker, English text PDFs, 10 papers, 20 pages)
- [x] **3. Story structure**
  - [x] 3.1 Define the epics or groups following the chosen approach
  - [x] 3.2 Write the shared constraints list that applies to every story
- [x] **4. Stories** (`aidlc-docs/inception/user-stories/stories.md`)
  - [x] 4.1 Write each story as "As a [persona], I want [goal], so that [benefit]"
  - [x] 4.2 Write acceptance criteria in the chosen format, covering the main path and the failure and edge cases chosen in Question 5
  - [x] 4.3 Make sure criteria about model output follow the answer to Question 7
  - [x] 4.4 Add requirement IDs, the "Demo path" tag and the "Depends on" field
- [x] **5. Check quality**
  - [x] 5.1 Check every story against INVEST (Independent, Negotiable, Valuable, Estimable, Small, Testable), and split or merge stories that fail. A story counts as Independent if all its hard dependencies are listed in "Depends on", it can be tested without running any other story first (using seeded data or the fake model), and the dependencies form no loops
  - [x] 5.2 Check that every acceptance criterion can be tested with the fake model, `TestClient`, the evaluation script or the manual checklist
  - [x] 5.3 Check the coverage table: every FR, NFR and SC row points to a story, the shared constraints list, or "Team practice: checked in Build and Test"
- [x] **6. Persona mapping**
  - [x] 6.1 Add a table in `personas.md` mapping each persona to its stories
- [x] **7. Finish**
  - [x] 7.1 Update `aidlc-state.md` and `audit.md`
  - [x] 7.2 Present the completion message for review
