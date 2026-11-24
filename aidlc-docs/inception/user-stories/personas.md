# Personas: Research Tracker

Two personas, as decided in the story plan (Q1: D, Q2: A). The MVP has one kind of end user. Undergraduates, graduate students and research assistants all work within the same limits, use the same screens and follow the same journey, so a single end-user persona represents all of them. The second persona is a team member checking the app against the success criteria.

---

## P1: Student Researcher

**Who**: An undergraduate or graduate student, or a research assistant, starting a literature review or research paper on one question, for example "How does social media use affect adolescent mental health?" [V §4, V §5]

**Context**
- Reads papers over several days or weeks, a few at a time
- Works alone, on their own laptop, often offline or on shared Wi-Fi
- Comfortable with a terminal. Can clone a repository and follow a README to run `ollama` and `uv` commands, but does not want to debug Python [Q2]
- Installs and starts the app themselves [Q1]

**Goals**
- See what each new paper adds to what they have already read, without rereading earlier notes
- Find where papers agree, contradict or overlap, and which subtopics they have covered
- Trust what the tool says by checking the passage and page behind every claim
- Pick up where they left off on another day without uploading papers again
- Take a readable summary into their own writing

**Frustrations today**
- Notes on separate papers do not show how findings relate
- Remembering which paper said what, and on which page, means opening PDFs again
- AI summaries that sound confident but cannot be checked

**What success looks like**
- Every label comes with a reason and passages they can open
- Quotes marked verified really are on the cited page, and failed checks are clearly flagged
- Nothing they saved disappears or changes without explanation

**MVP limits they work within**
- One tracker and one research question [Q8]
- English, text-based PDFs only; no scanned papers [V §2, A-1]
- Up to 10 papers per tracker and 20 PDF pages per paper [V §2]
- One upload at a time; processing may take minutes on a laptop [CQ5, NFR-2.1]
- Labels and sections can be inspected but not edited, and papers cannot be removed [V §3, Q9]

---

## P2: Team Evaluator

**Who**: A member of the project team checking whether the app meets the success criteria (SC-2, SC-3, SC-5), and tuning settings such as thinking mode and chunk size. [Q17, CQ6, CQ7]

**Context**
- Has the repository, the labeled test set of up to 10 papers, and the team's manual labels
- Runs commands from the terminal and reads code
- Re-runs the evaluation each time a setting changes [Q17]
- Works on their own laptop, whose speed differs from teammates' laptops [Q12]

**Goals**
- Measure label agreement, quote accuracy and count accuracy without checking each paper by hand
- Confirm independently, by eye, that quotes marked verified are on their cited page [FR-17.6]
- Find wiring bugs where the shown result differs from what was saved [FR-17.7]
- See which processing step is slow on their laptop [CQ7]
- Compare settings fairly, with nothing left over from earlier runs [CQ6]

**Frustrations to avoid**
- Test runs mixing papers into a real tracker
- Results that pass only because the app checks itself
- Slow manual re-checking every time a setting changes

**What success looks like**
- One command produces a report and a quote checklist
- Results can be opened in the web page to see why a paper got the wrong label

---

## Persona-to-Story Map

| Persona | Stories |
|---|---|
| P1 Student Researcher | US-01 to US-23 |
| P2 Team Evaluator | US-24 to US-29 |

| Epic | Stories | Persona |
|---|---|---|
| E1 Set up and start | US-01, US-02 | P1 |
| E2 Create a tracker | US-03 | P1 |
| E3 Upload a paper | US-04 to US-08 | P1 |
| E4 Read the card and label | US-09 to US-15 | P1 |
| E5 Inspect evidence | US-16 to US-18 | P1 |
| E6 See the tracker and its coverage | US-19 to US-21 | P1 |
| E7 Export and return later | US-22, US-23 | P1 |
| E8 Evaluate the app | US-24 to US-29 | P2 |
