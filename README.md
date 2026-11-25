
# Research Tracker

## Team Members

- Nicole Lin (njl55)
- Monisha Bommu (mrb359)
- Claire Cheng (cqc6)
- Amber Wang (aw896)
- Tianran Song (ts667)

**Coding agent(s):** Claude Code, ChatGPT Codex  
**Development methodology:** AI-Driven Development Lifecycle (AI-DLC), v1.0.1  
**Model:** Qwen3 8B  
**Runtime/provider:** Ollama, running locally  
**API:** [Ollama's OpenAI-compatible API](https://docs.ollama.com/openai)

## 1. Problem and Rationale for Using LLMs

Research Tracker helps students and early-stage researchers organize papers around a research question as their reading develops. Existing notes often describe papers individually, leaving readers to manually remember how findings relate, which perspectives they have covered, and where additional reading would help. Our product maintains a persistent, evolving research tracker that explains what each newly added paper contributes to the collection. We chose this problem because comparing sources and maintaining a coherent picture of a topic are recurring challenges in coursework and research for both students and faculty.

LLMs are appropriate for this problem because useful comparisons require interpreting meaning across papers that may use different terminology, methods, and structures. Keyword matching alone cannot reliably determine whether a paper introduces a new perspective, provides additional evidence, or challenges an earlier finding. The LLM will extract findings relevant to the user’s research question and explain relationships between sources. Its assessments will include supporting passages and page references so users can inspect the evidence. The tracker will distinguish uncertainty from supported conclusions and describe coverage within the uploaded collection rather than claiming to represent the entire research literature.

## 2. Planned LLM Integration

Research Tracker will use Qwen3 8B, running locally through Ollama and accessed through its OpenAI-compatible API. After a user uploads a paper, application code will extract its text, preserve PDF page numbers, and divide it into manageable chunks. The agent will process these chunks to create a structured paper card containing publication information, relevant findings, methods, limitations, and one or two supporting quotations with page references. It will compare the new card with existing paper cards in small batches, retrieving original passages when needed to assess relationships without loading the entire collection into one prompt. The agent will classify the paper as introducing a new angle, adding to an existing section, or substantially overlapping with an earlier paper. For additions, it will explain whether comparable findings agree, contradict, or extend earlier work, or indicate that the relationship is unclear. The comparison agent uses read-only tools to search passages, read saved cards, and list sections. The application saves the tracker updates and maintains the summary across sessions. Application code will handle PDF extraction, quotation and page-reference checks, storage, and counts, while the LLM will handle semantic interpretation and comparison.

## 3. Project Scope and Feasibility

By the December 16 deadline, our MVP will support creating a tracker from a research question, uploading English text-based PDFs individually, generating cited paper cards, classifying contributions, and updating a saved `tracker.md` summary. The MVP will support up to 10 papers per tracker and 20 PDF pages per paper. Users will be able to reopen their tracker across sessions and inspect the evidence behind classifications. The product will show paper counts by section and the publication-year range of the collection. We will keep automated paper discovery, scanned-PDF processing, reference-manager integrations, and collaborative editing out of scope. This scope combines document processing, evidence retrieval, cross-paper reasoning, and persistent state into a complete workflow. We will evaluate it using a small, manually reviewed paper collection, checking citation accuracy, classification quality, and whether saved updates preserve previous information.

## 4. Collaboration Plan

Our team will meet once every 2–4 days on Zoom to review progress, resolve design questions, and approve AI-DLC outputs. Between meetings, we will communicate through Slack and track tasks and decisions in GitHub issues and the project repository. We will divide implementation across five primary areas: PDF processing and retrieval, paper-card generation, comparison and classification, tracker summaries and coverage, and interface development and persistence. Each area will have one primary owner and a teammate who reviews its changes. We will agree on shared data formats and tool interfaces before implementation, use sample inputs to support parallel development, and share responsibility for integration, evaluation, and documentation. We will review workloads regularly and redistribute tasks as needed.


