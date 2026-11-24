
# Vision: Research Tracker

**AI-DLC:** v1.0.1  
**MVP deadline:** December 16, 2026  
**Team:** Nicole Lin (njl55), Monisha Bommu (mrb359), Claire Cheng (cqc6), Amber Wang (aw896)

## 1. Executive Summary

Students often record academic papers as separate summaries, making it difficult to remember how findings relate as their reading grows. Identifying overlapping evidence, conflicting conclusions, and gaps in their collection requires repeatedly revisiting earlier notes. Research Tracker helps students organize academic papers around a single research question. Each time the user uploads a paper, an LLM agent writes a cited paper card. It then compares the paper with the ones already in the tracker and labels it as a new angle, an addition to an existing section, or redundant. The agent chooses which source passages to retrieve when investigating relationships and supports its classifications with evidence. The app preserves paper cards, sections, citations, and source evidence across sessions. Users can inspect the tracker in the interface and export a readable Markdown summary. All assessments describe only the papers the user has uploaded, not the research field as a whole.

## 2. Features In Scope (MVP)

- **Create a tracker:** the user enters a research question, and the app creates a saved tracker with the question, an empty summary, and no sections yet.
- **Upload papers one at a time:** English, text-based PDFs only, with an initial scope of up to 10 papers per tracker and 20 PDF pages per paper. The app extracts the text with page numbers and shows a clear error if a PDF cannot be read, has insufficient extractable text, or exceeds these limits.
- **Process papers in chunks:** the app divides extracted text into manageable chunks with page references. The LLM processes these chunks to build a paper card instead of receiving the entire paper collection in one prompt.
- **Paper cards:** title, authors, and year, marked "unavailable" if missing or uncertain, findings relevant to the question, methods, limitations, and 1–2 supporting quotes with PDF page numbers. Findings retain references to supporting source passages.
- **Quote checks:** code confirms each quote appears on its cited page after normalizing whitespace and line-break hyphenation. Quotes that fail the check are flagged, not shown as verified. Verification confirms the quotation’s location, not the correctness of its interpretation.
- **Page-number convention:** citations use 1-based PDF page numbers, labeled "PDF page." These may differ from page numbers printed within the paper.
- **Classification:** each relevant paper gets one primary label and belongs to one primary section:
  - **New angle:** starts a new section because it introduces a relevant subtopic, method, or viewpoint not represented in the existing sections.
  - **Adds to [section]:** contributes additional relevant evidence, with a relationship of agrees, contradicts, extends, or unclear. Independent evidence supporting an existing finding can count as an addition.
  - **Redundant with [paper]:** no substantial new contribution to the research question is identified beyond an existing paper. It is still recorded in the referenced paper’s section, not discarded. This label describes overlap within the uploaded collection, not the paper’s overall value.
- **Evidence-based comparison:** the agent compares the new card against existing cards in small batches and retrieves original passages when needed. Agreement or contradiction is assessed only when claims are sufficiently comparable in methods, populations, and conditions.
- **Reasons:** every label comes with a one-sentence explanation and references to the passages it relied on. Cross-paper assessments reference evidence from both papers.
- **Off-topic papers:** papers with no established connection to the question are recorded separately as "Not relevant or relevance unclear," with a reason. They are excluded from topical section counts but included in the total saved-paper count.
- **Duplicate detection:** code catches an identical PDF uploaded twice before it reaches the model. Detecting different files or versions of the same paper is out of scope.
- **Tracker updates:** add the card, create or update the section, and rewrite the short overview. Earlier paper cards and their citations are preserved, while section summaries and the overview may be updated.
- **Coverage counts:** paper counts per section and the earliest and latest known publication years across the saved collection. Redundant papers count toward their assigned section, with their number also shown separately. Papers with unavailable years are reported separately.
- **Persistence:** close and reopen the tracker without losing cards, sections, citations, or access to source evidence. Users do not need to upload earlier papers again.

## 3. Features Explicitly Out of Scope (MVP)

- Imbalance notes and suggested reading topics (Phase 2)
- Manual editing of sections and classifications (Phase 2)
- Scanned or image-only PDFs (Phase 2; reliable quote checks need extractable text)
- Searching academic databases or recommending specific papers (Phase 3)
- Reference-manager integrations, e.g., Zotero (Phase 3)
- Non-English papers and non-PDF formats (Phase 3)
- Accounts, cloud sync, and shared trackers (Phase 3)
- Internet search for additional papers (Phase 3)
- Generating a full literature review (not planned; the tool supports the user's judgment rather than replacing it)

## 4. Target Users

- Undergraduate and graduate students writing literature reviews or research papers
- Early-stage researchers and research assistants exploring an unfamiliar topic

The MVP supports one user, one research question, and a locally run app.

## 5. Core User Journey

A student enters "How does social media use affect adolescent mental health?" and gets an empty tracker. The first paper they upload produces a cited card and a new section. Each later upload is compared with the earlier papers, labeled with a reason and supporting passages, and added to the summary. After several uploads, the student checks the section counts and year range, inspects the evidence behind each label, and reopens the tracker the next day to keep going.

## 6. Key Success Metrics

Measured on a manually labeled test set of up to 10 papers on one question, including redundant and contradicting pairs:

- End-to-end demo: create a tracker, add papers, inspect the evidence, then reopen it and add another paper
- 100% of quotes shown as verified appear on their cited page
- At least 80% of primary labels match the team's manual labels
- No earlier cards or sections lost after saving, reopening, and adding a paper
- Section counts and year range match the saved collection exactly

## 7. Risks

- **PDF page numbers:** the page number printed on a paper often differs from the PDF's page index, and multi-column layouts can scramble the extracted text. We need one consistent page-number convention.
- **Subjective labels:** papers can partly overlap while differing in methods, populations, or conclusions. That's why every label has a reason and "unclear" is an allowed relationship.
- **Overwriting earlier notes:** rewriting the tracker after each upload could drop older content. This needs testing across many additions and sessions.
- **Local model limits:** Qwen3 8B may cut off long papers if the context window is too small, return badly formatted output, or slow down as the collection grows.

## 8. Open Questions

- Can a paper belong to more than one section? (This decides how counts and redundancy work.)
- Where is the line between "redundant" and "adds modest evidence"? Do redundant papers count toward section totals?
- If a label or section name is wrong, is inspecting it enough for the MVP, or do users need a simple way to correct it?
- How should citations appear in the tracker and in the inspection view, and what does the user see when a quote can't be verified?
- How should an off-topic paper be handled: a "not relevant" label, or rejected at upload?
- Is Qwen3 8B accurate and fast enough on our laptops? What context window do we need for long papers, and what is the fallback if it falls short?
- Will the interface be a command-line tool or a local web page?
