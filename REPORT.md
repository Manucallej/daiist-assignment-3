# Assignment 3 Report

- **Name**: Manuela Calle
- **Student ID**: **TODO (you)**
- **Group**: **TODO (you)** [BBADBA 5A | BBADBA 5B | PPLEDBA 5A]

> Draft status: the facts and numbers below come from the code and from `eval/`. Sections marked **TODO (you)** need your own words or your own work: your grades, the failure analysis, the decision, and the subagent transcript. Please read everything and rewrite what you do not agree with, since the written check in class is about your choices.

## Use case

The user is a hotel manager or compliance officer at a mid-size hotel group who has to decide whether a guest-facing AI tool is allowed: a booking chatbot, a tool that personalises offers, emotion analysis of guest reviews, CV screening for hiring. They ask things like "do we have to tell guests they are talking to an AI?" and use the answer to decide what to build, buy or disclose.

A wrong answer is expensive: deploying a prohibited or non-compliant system risks fines of up to EUR 35 million or 7% of worldwide turnover (AI Act, Art. 99), and skipping a duty such as the chatbot disclosure risks fines of up to EUR 15 million or 3%. "I don't know" costs a call to a lawyer. So the assistant must answer only from the text, cite where, and refuse when the text does not cover the question.

**TODO (you):** adjust this to your own framing if you see the user differently.

## Corpus

| Document | URL | Date | Licence | Words indexed |
|---|---|---|---|---|
| `ai_act`: Regulation (EU) 2024/1689 (AI Act), articles and annexes | https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=OJ:L_202401689 | 2024-06-13 | EU legal act, reuse under Commission Decision 2011/833/EU (CC BY 4.0) | 52,242 |
| `gdpr`: Regulation (EU) 2016/679, articles | https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32016R0679 | 2016-04-27 | same | 29,682 |
| `ai_system_definition_guidelines`: Commission Guidelines on the definition of an AI system, C(2025) 5053 | https://ec.europa.eu/newsroom/dae/redirection/document/112455 | 2025-07-29 | Commission document, same reuse decision | 5,455 |
| **Total** | | | | **87,379** |

Added later in the add-document demo: `prohibited_practices_guidelines`, Commission Guidelines on prohibited AI practices, C(2025) 5052, 60,706 words (https://ec.europa.eu/newsroom/dae/redirection/document/112367).

The documents contain no personal data. I left out the recitals of both regulations (about 37,000 words in the AI Act and 26,000 in the GDPR): they are not binding, and the embedding quota of the free tier (about 1,000 texts a day) would not have covered two chunking designs over the full text. This is a limitation: questions whose answer only sits in a recital cannot be answered. **TODO (you):** please confirm the licence wording against the EUR-Lex and Commission reuse pages before submitting.

## Ingestion

`index.py` downloads the PDFs into `corpus/` (git-ignored), then for each file:

- **Extract** with `pypdf`, page by page, keeping the PDF page number.
- **Clean**: remove the Official Journal headers and footers (`OJ L, 12.7.2024 EN`, `ELI: http://...`, `L 119/41 Official Journal ...`), lone page numbers, table-of-contents lines, footnotes that cite the Official Journal, soft hyphens, and everything before "HAVE ADOPTED THIS REGULATION" (the recitals).
- **Split into units** with regular expressions: `Article N`, `ANNEX X` and, for the guidelines, `Part I` and numbered sections. Within a unit, paragraphs are found by their numbers (`1.`, `(12)`).
- **Chunk** in two designs (below), embed with `gemini-embedding-2` (768 dimensions, normalised), and store in ChromaDB under `index/chroma`. Each batch is written as soon as it is embedded, and only chunks without a stored vector are embedded, so a crash or the daily quota loses nothing. Query vectors are cached in `index/query_cache.json`.

One extraction problem I fixed: the Official Journal page header and footer lines landed in the middle of articles and, in the GDPR, broke the "Article N" lines that the structure detection depends on. The line patterns above remove them. A second one: words split by soft hyphens in the PDF (`internat\xad ional`) are rejoined.

**Metadata per chunk:** `source`, `title`, `design`, `unit` (e.g. Article 50), `units` (all units a fixed-size chunk touches), `location` (e.g. "Article 50(1)-(3) - Transparency obligations ..."), `page`, `page_end` (PDF pages), `chapter`, `n_words`.

**Chunks:** fixed-size 291 (174 AI Act, 99 GDPR, 18 guidelines); structure-based 345 (196, 131, 18). 636 embeddings in total, 21.6 MB in `index/`.

Known weakness: guideline footnotes are only partly removed, and headings that wrap over two lines are cut after the first line.

## Test set

I drafted 35 questions from the documents before running any retrieval or agent: 22 `single`, 6 `multi` (two passages, sometimes from two documents), 7 `unanswerable`. Single and multi questions have an expected source and article, annex or section; `reference_answer` is a short correct answer. Unanswerable questions are close to the topic (Spanish authority, Spanish age of consent, enforcement statistics, specific technical standards, the ePrivacy Directive, Spain's sandbox, and one that only the later-added guidelines answer).

Examples:

- `q01` (single): "Do we have to tell guests that they are talking to an AI chatbot?" Expected: `ai_act`, Article 50.
- `q23` (multi): "Our booking chatbot collects guests' personal data. What do the AI Act and the GDPR each require us to tell guests?" Expected: `ai_act` Article 50 and `gdpr` Article 13.
- `q30` (unanswerable): "At what age can a minor in Spain consent to a hotel processing their data?" Expected: the documents give only the GDPR default of 16 and the 13 floor, not Spain's choice.

**TODO (you):** the assignment says the test set must be written by you. Read all 35, rewrite or replace the ones you would not ask, and check each reference answer against the text. Say here how you actually wrote it.

## Retrieval

28 answerable questions; a question is a hit at k if every expected source and article is among the top k chunks. Words are the average number of words the top 5 chunks add to the prompt.

| Design | hit@1 | hit@3 | hit@5 | words in prompt (k=5) |
|---|---|---|---|---|
| A. fixed 350 words, overlap 50, vector | 0.679 | 0.821 | 0.857 | 1,748 |
| A. fixed, hybrid (vector + BM25, rank fusion) | 0.571 | 0.750 | 0.857 | 1,750 |
| B. structure-based (one chunk per article/section, split at paragraphs over 450 words), vector | 0.714 | 0.857 | **0.964** | 1,723 |
| B. structure-based, hybrid | 0.607 | **0.929** | **0.964** | 1,754 |

Chosen: **B structure-based + hybrid**, by the rule "highest hit@5, then hit@3, then hit@1". Structure-based chunking is clearly better at k = 5 (+11 points) and keeps the article in the chunk header. Hybrid ties vector at k = 5 and wins at k = 3, but is worse at k = 1 (0.607 against 0.714); the agent sees five passages, so I chose by k = 5 and k = 3. With 28 questions, one question is 3.6 points, so these differences are small.

**TODO (you):** say whether you agree with this rule, and what you would change.

## Tools and subagent

`server.py` (MCP server `corpus`) has two tools. The model reads these descriptions:

- **search(query)**: "Search the EU AI Act, the GDPR and the Commission guidelines on the AI system definition (plus any document added later) and return the 5 most relevant passages. Use it for any question about what these EU texts say ... Write the query as a short, specific description of the rule you need ... Returns numbered passages. Each has its citation in square brackets (source, article / annex / section, page), a relevance score and the text. Cite with exactly that bracket. These texts are EU law only: national laws and guidance from other bodies are not in the index, so an empty or off-topic result means the corpus does not cover the question."
- **add_document(path)**: "Add one document to the searchable index: reads the file at `path`, splits it into chunks, embeds them and stores them ... If the very same file is already indexed, nothing is changed and the tool says so. If a file with the same name but different content is indexed, its old chunks are replaced ... This changes the shared index: use it only when the user asked to add a document."

One search result (first passage, shortened):

```
[1] cite as [ai_act, Article 50(1)-(3)] | AI Act (Regulation (EU) 2024/1689) | Article 50(1)-(3) - Transparency obligations for providers and deployers of certain AI systems | PDF page 82 | score 0.032
AI Act | Chapter IV - Transparency Obligations For Providers And Deployers Of Certain AI Systems | Article 50(1)-(3) - ...
1. Providers shall ensure that AI systems intended to interact directly with natural persons are designed and developed in such a way that the natural persons concerned are informed that they are interacting with an AI system, unless this is obvious ...
```

(The score is the rank-fusion score of the hybrid search.)

**Subagent** `.claude/agents/corpus-researcher.md`: tools limited to `mcp__corpus__search`. Instructions: search before answering (and again with other terms if needed, once per rule for multi-part questions); answer only from the passages; cite every claim with the bracket as given; keep conditions and exceptions; say so if the documents do not cover the question; be short and end with "This is a reading of the text, not legal advice."

**TODO (you):** open the repo folder in Claude Code or VS Code Copilot, check that the `corpus` server shows its two tools, ask the subagent a question (for example q01 or one of the unanswerable ones) and paste the transcript here. I cannot produce this from my side, and the assignment wants a real one from your editor.

## Adding a document

If the same file (same SHA-256) is already indexed, `add_document` embeds nothing and reports "Already indexed ... Nothing was changed." If a file with the same name but different content is indexed, its old chunks are deleted and the new ones are indexed (the tool says "Replaced"). The tool only writes to the design the server uses (structure-based), so the other design is not kept up to date. Only the chunks without a stored vector are embedded.

Demo (`eval/add_document_demo.json`), unanswerable question `q34`: "Does the ban on emotion recognition at work also cover job candidates during hiring?" The AI Act text says only "workplace and education institutions".

- **Before:** the assistant said the indexed documents "do not explicitly state" whether candidates are covered, then listed Article 5(1)(f) and the high-risk employment area in Annex III as related rules. A hedged answer rather than a clean refusal; the judge marked it incorrect.
- **Added:** `prohibited_practices_guidelines` (60,706 words, 222 chunks) through the `add_document` tool, with approval. Adding the same file again returned "Already indexed. Nothing was changed."
- **After:** "Yes, the ban on emotion recognition in the workplace also covers job candidates", citing `[prohibited_practices_guidelines, Section 7.2.2, para (254)]` and the statement that "using emotion recognition AI systems during the recruitment process is prohibited". I checked both against the PDF: paragraph (254) says the notion of workplace applies to candidates during selection and hiring, and the example list contains that sentence.

The evaluation removes the added document afterwards, so the committed index holds the 3-document baseline.

## Harness and guardrails

`agent.py` uses LangChain `create_agent` with the two MCP tools and `AGENTS.md` as the system prompt. The rules in `AGENTS.md` (each has its reason in the file): answer only from the search results; cite every claim with the bracket from the result; say when the corpus does not cover the question; search again and combine passages before giving up; stay with what the text says including its exceptions; no legal advice; add documents only on request; be short.

- **Approval before add_document:** the tool wrapper calls `approve(path)` before running the real tool. From the command line it asks "Approve? [y/N]"; with no terminal it refuses. Case: `main.py ask "Please add the file extra/prohibited_practices_guidelines.pdf to the index."` printed `[approval] add_document(...) refused: no terminal to ask.`, the assistant reported that the file was not added, and `index/documents.json` was unchanged.
- **Call limit:** at most 6 searches per question; after that the tool tells the model to answer or say the corpus does not cover it.
- **Extra guardrail, citation check:** every `[source, Article N]` in the answer must match a passage that a search returned. If not, the model gets one chance to fix it; if it still cites something it never received, those citations are replaced by "[citation removed: not in the search results]" and a warning is added. In the 35 evaluation questions it did not fire (0 cases), because the model only cited what it had received. **TODO (you):** the assignment asks for one case where the extra guardrail fired. If you want a real one, try a prompt that pushes the model to cite from memory (for example "answer without searching and cite the article"), or switch the guardrail to one that fired in your own tests.

## Evaluation

**Model note:** the free Gemini tier allows about 20 generation requests a day per Gemini model, which cannot cover 35 agent runs plus judging. The agent therefore runs on `gemma-4-31b-it` and the judge on `gemma-4-26b-a4b-it` (a different model from the agent). Both are set by `AGENT_MODEL` and `JUDGE_MODEL`. A stronger judge would be better; the numbers below are only as good as this one.

**Code checks** (`evaluate.py`): citations valid (each cited source and article was retrieved); refusal detected on unanswerable questions; wrong refusal on answerable ones (refusal wording and no citation); expected passage retrieved.

**Judge rubric:** `correct`: for an answerable question the answer states the key facts of the reference and contradicts none (a refusal is 0); for an unanswerable one it says the indexed documents do not cover it. `supported`: every claim is in the retrieved passages; claims that are true but not retrieved count as 0. The judge sees the question, the reference answer, the retrieved text and the answer.

**Agreement with the judge:** **TODO (you).** `eval/answers_for_grading.csv` has the questions, reference answers and the assistant's answers, without any verdicts. Grade at least 20 in `eval/human_labels.csv` (`id`, `correct`, `supported`, `notes` for every 0) before opening `eval/results.csv`, then run `uv run python main.py evaluate` again: it prints your agreement and Cohen's kappa with the judge in `eval/summary.json`.

**Results (35 questions, judge verdicts, not yet checked against your grades):**

| | |
|---|---|
| Correct | 34 / 35 |
| Answerable questions correct | 28 / 28 |
| Unanswerable correctly refused | 6 / 7 |
| Wrong refusals (answerable, refused) | 0 |
| Unsupported claims (judge) | 0 |
| Answers with an invalid citation | 0 |
| Guardrail fired | 0 |

## Failures

One question failed, `q34` (the emotion recognition and candidates question). The assistant did not give a made-up answer: it said the documents do not explicitly cover it, and then listed related rules. The judge marked it incorrect because it did not refuse cleanly. This could be counted as "unsupported claim" (the automatic label), as a judge that is too strict, or as a weak test question, since the AI Act text does touch the topic. **TODO (you):** decide how you classify it, and give your count by cause (retrieval miss, wrong answer from the right passage, unsupported claim, wrong refusal, bad test question).

Honest warning about the numbers: 34 of 35 is a very good result and a small, fairly easy test set. Most questions ask for one article by topic. I would not read this as proof that the assistant is reliable. What to fix first: write harder questions (conditions and exceptions, questions where two articles seem to conflict, and unanswerable ones that sound answerable), and grade a sample by hand, since the judge is a small model.

## Decision

**TODO (you):** would you let users rely on it, for what, and with which safeguard? Facts to use:

- On this test set it answers correctly and cites valid passages almost every time, and it refused 6 of 7 unanswerable questions.
- It is a research aid over three EU texts only. Recitals, national laws (for example Spain's), case law, other EU laws and anything newer than the documents are not covered, and the AI Act is being amended and guided over time.
- Cost per question on `gemma-4-31b-it`: about 5,900 tokens in and 650 out on average (1 to 3 searches); latency 111 s on average and 178 s at the 90th percentile, mostly rate-limit waiting on the free tier. In my first nine runs on `gemini-3-flash-preview` it took 4 to 10 s per question.
- Embedding the corpus (636 chunks) fits in one day of the free tier; the add-document demo needed another 222 embeddings.

## AI use

I used Claude (Claude Code) to write the code (`index.py`, `server.py`, `agent.py`, `evaluate.py`), to draft `AGENTS.md`, the subagent file, the 35 test questions with reference answers, and this report's factual parts. The use case, the choice of corpus, the design decision, the test questions, the rubric, my own grades and the analysis and decision are mine. **TODO (you):** confirm that this is accurate and change it to what is true for your own work; the assignment requires the disclosure and requires the decisions and analysis to be yours.
