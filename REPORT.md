# Assignment 3 Report
 
* Name: Manuela Calle
* Student ID: 18754
* Group: BBADBA 5A
 
## Use case
 
The users are hotel managers and compliance officers who need to decide whether a guest-facing AI tool is allowed under EU law. Examples are a booking chatbot, personalised offers, emotion analysis of reviews and CV screening. They use the answer to decide what AI to build, buy or disclose to guests. The question can start with the manager, but the decision has to hold up later in front of an auditor, a regulator, a shareholder or an unhappy guest, and it may reach the CEO or the chief data officer.
 
A wrong answer is expensive. Deploying a prohibited practice can mean fines of up to 35 million euros or 7% of worldwide turnover, and skipping a duty such as chatbot disclosure up to 15 million euros or 3% (AI Act, Article 99). An "I don't know" when the system is unsure, the cost is time and a call to a lawyer to clear doubts. Because of this gap, the system is built to answer only from the text, cite where, and refuse when it's unsure of the answer.
 
## Corpus
 
| Document | URL | Date | Licence | Words |
|---|---|---|---|---|
| AI Act (Regulation (EU) 2024/1689) | https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=OJ:L_202401689 | 2024-06-13 | Reuse under Commission Decision 2011/833/EU (CC BY 4.0) | 52,242 |
| GDPR (Regulation (EU) 2016/679) | https://eur-lex.europa.eu/legal-content/EN/TXT/PDF/?uri=CELEX:32016R0679 | 2016-04-27 | Reuse under Commission Decision 2011/833/EU (CC BY 4.0) | 29,682 |
| Commission Guidelines on the definition of an AI system (C(2025) 5053) | https://ec.europa.eu/newsroom/dae/redirection/document/112455 | 2025-07-29 | Reuse under Commission Decision 2011/833/EU (CC BY 4.0) | 5,455 |
| **Total** | | | | **87,379** |
 
All three are public EU documents with no personal data. I left out the recitals (the long "whereas" introductions). They are not legally binding, and the free embedding limit of about 1,000 texts a day could not cover two chunking designs over the full text. The cost is that anything explained only in a recital cannot be answered.
 
## Ingestion
 
I extract the text from each PDF page by page and keep the page number as a label. Then I clean it by removing Official Journal headers and footers, printed page numbers, tables of contents, footnotes, soft hyphens and the recitals. The page number stays in the metadata even though it is removed from the text.
 
One extraction problem was that page headers landed in the middle of articles and broke the "Article N" lines that the structure detection depends on. To solve this, I created a file with the pattern that was seen in the corpus when this happened, and then those patterns that matched were deleted in the cleaning.  
 
There are two chunking designs. Design A cuts fixed windows of 350 words with 50 words of overlap. Design B cuts by structure, one chunk per article or section, and only splits at paragraph breaks if the unit is longer than 450 words. Each chunk carries this metadata: source document, article, annex or section, page, chapter and word count, and that metadata becomes the citation. Design A has 291 chunks (AI Act 174, GDPR 99, guidelines 18) and Design B has 345 chunks (AI Act 196, GDPR 131, guidelines 18). In total 636 chunks were embedded with Gemini. We mainly used Design B, because logically if we discuss legal information where citations and exact language is important, it's important for information not to be cut off and for the RAG to be able to find the FULL citation depending on the text's natural structure. 
 
## Test set
 
I wrote 35 questions from the documents before tuning anything. Each has an expected source and location and a short reference answer. There are 22 where one passage answers it, 6 where two passages are needed, sometimes from two documents, and 7 unanswerable close to the topic but not in the corpus, for example Spain's data authority or the Spanish age of consent.
 
Examples:
* q01 (single): "Do we have to tell guests that they are talking to an AI chatbot?" Expected source is AI Act Article 50.
* q23 (multi): "Our booking chatbot collects guests' personal data. What do the AI Act and the GDPR each require us to tell guests?" Expected sources are AI Act Article 50 and GDPR Article 13.
## Retrieval
 
Hit@k is the share of the 28 answerable questions where the right passage is in the top k results. For multi questions every needed passage must appear.
 
| Design | hit@1 | hit@3 | hit@5 | words in prompt at k=5 |
|---|---|---|---|---|
| A fixed, vector | 0.679 | 0.821 | 0.857 | 1,748 |
| A fixed, hybrid | 0.571 | 0.750 | 0.857 | 1,750 |
| B structure, vector | 0.714 | 0.857 | 0.964 | 1,723 |
| B structure, hybrid | 0.607 | 0.929 | 0.964 | 1,754 |
 
I chose by highest hit@5, then hit@3, then hit@1, because the agent reads five passages. The winner is **structure-based chunks with hybrid search**. Structure beat fixed-size by 11 points at k=5. Hybrid is worse at hit@1 (0.607 against 0.714) but better at hit@3. With only 28 answerable questions one question is worth 3.6 points, so the differences between vector and hybrid are small and I should not oversell them. The five passages add about 1,750 words to the prompt for every design.
 
## Tools and subagent
 
`server.py` is an MCP server with two tools.
 
**search** (description the model reads): "Search the EU AI Act, the GDPR and the Commission guidelines on the AI system definition (plus any document added later) and return the 5 most relevant passages. Use it for any question about what these EU texts say... Returns numbered passages. Each has its citation in square brackets (source, article / annex / section, page), a relevance score and the text. Cite with exactly that bracket. These texts are EU law only: national laws and guidance from other bodies are not in the index, so an empty or off-topic result means the corpus does not cover the question."
 
**add_document** (description): "Add one document to the searchable index: reads the file at `path`, splits it into chunks, embeds them and stores them, so that later searches can cite it... If the very same file is already indexed, nothing is changed and the tool says so. If a file with the same name but different content is indexed, its old chunks are replaced by the new ones... This changes the shared index: use it only when the user asked to add a document."
 
One search result (query "transparency obligation AI system interaction natural persons chatbot", first of five):
 
```
[1] cite as [ai_act, Article 50(1)-(3)] | AI Act (Regulation (EU) 2024/1689) | Article 50(1)-(3) - Transparency obligations for providers and deployers of certain AI systems | PDF page 82 | score 0.033
1. Providers shall ensure that AI systems intended to interact directly with natural persons are designed and developed in such a way that the natural persons concerned are informed that they are interacting with an AI system, unless this is obvious ...
```
 
**Subagent.** `.claude/agents/corpus-researcher.md` has access only to the `search` tool, so it cannot change the index. Its instructions: search first with short legal queries and search again if needed, answer only from the passages received, cite every claim with the exact bracket, keep conditions and exceptions, say the documents do not cover it when they do not, be short, and end with "This is a reading of the text, not legal advice."
 
Transcript from my editor: [PASTE YOUR q01 TRANSCRIPT HERE]
 
Reflection: [YOU WRITE. Points from your notes: it searched once, cited Article 50(1)-(3), 50(4)-(7) and 26(11)-(12), all really retrieved. It noticed the disclosure duty falls on the provider, not the hotel as deployer. Weakness is that it did not check the GDPR side.]
 
## Adding a document
 
If the identical file (same SHA-256 fingerprint) is already indexed, nothing changes and the tool replies "Already indexed ... Nothing was changed." If a file has the same name but different content, the old chunks are replaced by the new ones. I tested the first case by adding the same file twice.
 
**Unanswerable question (q34):** "Does the ban on emotion recognition at work also cover job candidates during hiring?" The AI Act only says "workplace and education institutions".
 
**Document added:** the Commission Guidelines on prohibited AI practices (C(2025) 5052, https://ec.europa.eu/newsroom/dae/redirection/document/112367, 60,706 words, 222 chunks), added through the tool with approval.
 
**Before:** the answer said the indexed documents do not explicitly state whether the ban covers candidates, then listed related rules (Article 5(1) and the recruitment area of Annex III).
 
**After:** "Yes, the ban on emotion recognition in the workplace also covers job candidates." It cited the guidelines, Section 7.2.2, para (254), which says the notion of workplace applies to candidates during selection and hiring. Before the addition none of the retrieved passages came from this guideline. After it, Sections 7, 7.1, 7.2.2 and 7.2.3 of the new document were retrieved. The evaluation removes the document afterwards, so the committed index is the three-document baseline.
 
## Harness and guardrails
 
`agent.py` uses LangChain `create_agent` with `AGENTS.md` as the system prompt and the MCP tools from `server.py`.
 
**AGENTS.md rules** (each has its reason written next to it in the file): 1) answer only from search results, 2) cite every claim with the exact bracket and never cite what was not retrieved, 3) say when the corpus does not cover the question and stop, 4) search again before giving up and combine passages, 5) keep to what the text says including exceptions, 6) no legal advice, 7) add documents only on request, 8) be short.
 
**Approval step.** Before `add_document` runs, the tool asks the person "Approve? [y/N]". With no person present the answer is no. Reason: adding a document changes the shared index for everyone.
 
**Search limit.** At most 6 searches per question, as a safety net against loops, cost and the free-tier quota. Most questions needed one to three searches. The risk is that a limit that is too low causes a wrong refusal.
 
**Extra guardrail: citation check.** The code records every passage a search returned. Every `[source, Article N]` in the answer must match one of them. If not, the model gets one retry. If it still cites something it never received, the citation is replaced by "[citation removed: not in the search results]" and a warning is added. Reason: an invented legal citation is worse than none. It checks that the cited passage was retrieved, not that it supports the claim (the judge's "supported" grade covers that).
 
**Case where it fired:** in the real run of 35 questions it never fired (0 retries, 0 removals), because the agent only cited passages it received. [OPTION: add a deliberate test that calls `check_citations` and `strip_citations` on an answer citing an article that was never retrieved, and paste the output here, clearly labelled as a deliberate test and not a real case.]
 
## Evaluation
 
The agent (`gemma-4-31b-it`) answered all 35 questions. A different model (`gemma-4-26b-a4b-it`) acted as judge, so it does not grade its own work. I used Gemma because the free Gemini tier allows only about 20 generation requests a day.
 
**Code checks:** whether every citation matches a retrieved passage, whether the agent refused on unanswerable questions, and whether it wrongly refused an answerable one.
 
**Judge rubric** (two grades, 0 or 1):
* correct. For answerable questions, the answer states the key facts of the reference answer and contradicts none, and a refusal is 0. For unanswerable questions, 1 only if it says the indexed documents do not cover the question.
* supported. 1 if every factual claim is in the retrieved passages, even a true claim counts as 0 if the passages do not contain it. Closing lines like "not legal advice" do not count.
**Agreement with the judge.** I graded 26 answers myself before seeing the judge's verdicts. We agreed on 26 of 26 (100%) for both correct and supported. This is weaker evidence than it sounds. For "correct" the kappa of 1.0 rests on a single shared failure (q34). For "supported" we both gave 1 to all 26, so there is no variation and kappa is undefined (the script prints 1.0 as a default). High agreement on an easy set does not prove the judge would catch failures.
 
**Results (35 questions):**
* Correct: 34 of 35
* Answerable correct: 28 of 28
* Unanswerable correctly refused: 6 of 7
* Wrong refusals: 0
* Unsupported claims: 0
* Invalid citations: 0
## Failures
 
One question failed, q34. The agent did not refuse cleanly. It said the text does not define "workplace" and then listed related rules, which implied an answer. The judge and I both marked it incorrect.
 
| Cause | Count |
|---|---|
| Retrieval miss | 0 |
| Wrong answer from the right passage | 0 |
| Unsupported claim | [0 or 1, see below] |
| Wrong refusal | 0 |
| Bad test question | [0 or 1, see below] |
 
[YOU DECIDE. The evaluation script labelled q34 "unsupported claim (answered an unanswerable question)". The other reading is bad test question, because the AI Act does touch the topic (Article 5(1)(f) and Annex III) and my own reference answer admits the text is ambiguous. Pick one, count it once, and justify it in one or two sentences.]
 
What to fix first: [YOU WRITE. Options from your notes: add harder unanswerable questions that sound answerable and review q34's wording, then add questions with exceptions or conflicting articles.]
 
## Decision
 
[YOU WRITE. Facts to weigh: very accurate on this test set, but the set is small and fairly easy and most questions ask for one article by topic. Only three EU texts, no recitals, no national law such as Spain's, no case law, and the AI Act keeps getting new guidance (the q34 demo shows how one added document changed an answer). A reasonable position is to use it as a research aid that points to the right articles, with a person checking every citation and a lawyer for real decisions.]
 
**Cost and latency per question:** about 5,900 input tokens and 650 output tokens, with 1 to 3 searches. Average time was 111 seconds (90th percentile 178 seconds), mostly free-tier rate-limit waiting. On Gemini Flash it took 4 to 10 seconds.
 
## AI-use disclosure
 
[YOU WRITE. State honestly which parts you did yourself and which tools helped (for example Claude, Copilot) and for what.]
 
