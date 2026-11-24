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

[Answer]:

## Question 2

How technically comfortable should the end-user persona be assumed to be? This shapes the setup stories and how error messages are written.

A) Comfortable with a terminal: can clone a repository and run `ollama` and `uv` commands by following a README

B) Not technical: needs step-by-step setup instructions and plain-language error messages with no jargon

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 3

How should the stories be organized?

A) User journey only (see Part A)

B) Feature-based only

C) Epics ordered along the user journey, for example: Set up and start → Create tracker → Upload a paper → Read the card and label → Inspect evidence → See coverage → Export and return later → Evaluate the app

D) Epics that match the five ownership areas in the README (PDF processing and retrieval, paper cards, comparison and classification, summaries and coverage, interface and persistence)

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 4

How big should each story be?

A) Coarse: about 8–12 stories, roughly one per epic or major capability

B) Medium: about 20–30 stories, one per user-visible behavior (for example "see why a quote is not verified" is its own story)

C) Fine: 40 or more stories, including one per error case

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 5

How should failure and edge cases be handled? Examples: rejected uploads, an off-topic first paper, unverified quotes, failed processing, and a restart during processing.

A) As acceptance criteria inside the related main story

B) As separate stories when the user sees a distinct result (for example "upload is rejected" or "processing failed, try again"), and as acceptance criteria otherwise

C) Always as separate stories

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 6

What format should the acceptance criteria use?

A) Given / When / Then scenarios

B) A checklist of testable statements

C) Given / When / Then for behavior that involves the model or several steps; a checklist for simple rules (for example limits and counts)

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 7

Should acceptance criteria name the model's exact output, given that model results vary?

A) No. Criteria about model output check structure and rules only, never content. For example, "the label is one of the four allowed labels and has a one-sentence reason citing passages from both papers," not "the label is Adds to." Correct content is measured by the evaluation script (SC-3)

B) Yes. Use concrete expected labels for specific test papers

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 8

How should stories be traced back to the requirements?

A) Each story lists the FR, NFR and SC IDs it covers, and a table at the end of `stories.md` shows that every FR is covered by at least one story

B) Each story lists its IDs, but no coverage table

C) No ID references

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 9

Requirements such as localhost-only binding, autoescaping and SQL placeholders have no user-visible behavior. How should they appear?

A) As acceptance criteria on the story they protect (for example, autoescaping on the "read the card" story)

B) In a separate "constraints that apply to every story" list at the top of `stories.md`, and not repeated in each story

C) As their own stories written for the team evaluator persona

X) Other (please describe after [Answer]: tag below)

[Answer]:

## Question 10

Should each story carry a priority or demo tag?

A) Yes. Mark the stories needed for the SC-1 end-to-end demo as "Demo path," so the team can make sure those work first

B) Yes. Use MoSCoW (Must / Should / Could) within the MVP

C) No. Everything in the requirements is MVP scope, so there are no tags

X) Other (please describe after [Answer]: tag below)

[Answer]:

---

## Part C: Execution Checklist

This will run after the plan is approved. Steps that depend on your answers are worded generally for now.

- [ ] **1. Prepare**
  - [ ] 1.1 Re-read `requirements.md` and list every FR, NFR and SC ID
  - [ ] 1.2 Apply the answers from Part B (personas, organization, size, format)
- [ ] **2. Personas** (`aidlc-docs/inception/user-stories/personas.md`)
  - [ ] 2.1 Write each persona: goals, context, technical comfort, frustrations, what success looks like
  - [ ] 2.2 State the MVP limits each persona works within (one tracker, English text PDFs, 10 papers, 20 pages)
- [ ] **3. Story structure**
  - [ ] 3.1 Define the epics or groups following the chosen approach
  - [ ] 3.2 Write the constraints that apply to every story, if chosen in Question 9
- [ ] **4. Stories** (`aidlc-docs/inception/user-stories/stories.md`)
  - [ ] 4.1 Write each story as "As a [persona], I want [goal], so that [benefit]"
  - [ ] 4.2 Write acceptance criteria in the chosen format, covering the main path and the failure and edge cases chosen in Question 5
  - [ ] 4.3 Make sure criteria about model output follow the answer to Question 7
  - [ ] 4.4 Add requirement IDs and tags as chosen in Questions 8 and 10
- [ ] **5. Check quality**
  - [ ] 5.1 Check every story against INVEST (Independent, Negotiable, Valuable, Estimable, Small, Testable), and split or merge stories that fail
  - [ ] 5.2 Check that every acceptance criterion can be tested with the fake model, `TestClient`, the evaluation script or the manual checklist
  - [ ] 5.3 Check the requirements coverage (every FR covered at least once), if chosen in Question 8
- [ ] **6. Persona mapping**
  - [ ] 6.1 Add a table in `personas.md` mapping each persona to its stories
- [ ] **7. Finish**
  - [ ] 7.1 Update `aidlc-state.md` and `audit.md`
  - [ ] 7.2 Present the completion message for review
