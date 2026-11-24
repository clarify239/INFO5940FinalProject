# Story Plan Clarification Questions

All 10 answers are complete and consistent. Two of them leave a detail open that affects how the stories are written.

Fill in the letter after each `[Answer]:` tag. If no option fits, choose the last option (Other) and describe what you want.

---

## Ambiguity 1: Coverage table for NFRs that are not stories (Q8 + Q9)

Q8 extends the coverage table to NFRs and success criteria. Q9 puts the security rules in a shared constraints list instead of in stories. Some NFRs are neither user behavior nor shared runtime constraints. Examples are NFR-5 (prohibited libraries, Python 3.12, `uv.lock`) and parts of NFR-4 (pytest, a fake model, pull requests merged only when tests pass). These are rules about how the team builds the app, so no story naturally covers them.

### Clarification Question 1

In the coverage table, what can an NFR or SC row point to?

A) A story, the shared constraints list, or "Team practice: checked in Build and Test" for build and process rules. Every row must point to one of these three

B) Only a story or the shared constraints list. Build and process rules are added to the shared constraints list too, so everything is covered there

C) Track only FRs and SCs in the table, and leave NFRs out

X) Other (please describe after [Answer]: tag below)

[Answer]: A. The shared constraints list from Q9 holds runtime rules that every story's acceptance criteria must satisfy, like localhost binding and autoescaping. Rules like "Python 3.12" or "pull requests merge only when tests pass" aren't about how the app behaves, so they would make that list confusing (option B). Option C would undo the Q8 decision to put NFRs in the table. With A, every row says honestly where it gets checked, and the "Team practice" rows simply wait for the Build and Test stage, which is where AI-DLC checks those rules anyway.

---

## Ambiguity 2: What the user sees after a restart mid-processing (Q5)

Q5 makes "cleanup after a restart" an acceptance criterion rather than its own story. The requirements (FR-11.4) only say that partial data is removed on the next start. They do not say whether the user is told. As written, someone who closes the app while a paper is processing would reopen it to find the paper simply missing, with no explanation. Under your Q5 rule, a visible message would be a distinct result.

### Clarification Question 2

When the app starts and cleans up a paper that was interrupted, what should the user see?

A) A one-time notice on the tracker page, such as "[display name] was still processing when the app closed, so it was not saved. Upload it again to retry." This adds a small requirement (FR-11.6) and is an acceptance criterion on the "processing failed, try again" story

B) Nothing. The cleanup is silent, and the acceptance criterion only checks that the tracker is unchanged

X) Other (please describe after [Answer]: tag below)

[Answer]: A. If a paper silently disappears, it looks like the app lost data, and not losing data is what the vision's persistence goal and SC-4 are about. The cleanup in FR-11.4 already needs a "processing" marker for the paper, and that marker can hold the name for the notice. 
