# Design Write-up

**Assumptions:** (1) "last 5 years" is counted back from 2026, so installs before 2021 do not qualify; (2) the brief gives weights but no scoring scales, so I chose them (question 2); (3) a schedule over 16 weeks is flagged.

## 1. AI vs. rules

**AI only reads.** It turns each proposal's text into a list of facts (price, weeks, warranties, licence number, who does the electrical work, team items), each with a quoted excerpt and page. The list is built from the admin's criteria for the category, so nothing is hard-coded to solar. The AI never sees the weights and never scores or ranks.

**Code decides.** Code handles file checks, confirming each quote really appears in the document, mandatory pass/fail, scoring, sorting and logging.

AI is good at interpreting prose, but a decision affecting a contract must be identical every time and explainable. A fixed formula is, and text hidden in a proposal cannot steer it.

## 2. Consistency and explainability

**Scoring scales (my assumptions, stored in the criteria config):**
- Experience: 2 points per qualifying install (100 kW or more, last 5 years), capped at 15 installs = 30.
- Price: 25 points at 70% of the budget ceiling or lower, falling in a straight line to 0 at the ceiling.
- Timeline: 15 points at 10 weeks or fewer, falling to 0 at 16 weeks.
- Warranty: 7.5 for panels (25-year cap) plus 7.5 for workmanship (12-year cap), both in proportion.
- Team and approach: 3.75 each for a named lead, an identified crew, a safety plan and a monitoring plan.

**Same result twice?** Scoring is plain arithmetic, so the same approved facts and criteria version always give the same scores and order. Ties are broken by score, then lower price, then proposal letter. AI wording can differ between runs, so extraction uses temperature 0 and a pinned model version, and the reviewer-approved facts are saved and reused. If extraction is ever re-run, any changed fact is shown to the reviewer instead of silently replacing the old one.

**Why did it rank there?** For each proposal the reviewer sees every criterion with the fact used, its quote and page, the weight and the points.

**Sample result** (calculated with the scales above):

| | Exp. /30 | Price /25 | Time /15 | Warranty /15 | Team /15 | Total |
|---|---|---|---|---|---|---|
| A | 28 | 7.1 | 10 | 13.8 | 15 | **73.8** |
| E | 16 | 5.6 | 7.5 | 15 | 15 | **59.1** |
| C | 18 | 5.1 | 5 | 13.8 | 7.5 | **49.4** |

- **B** is ineligible: the licence is held by an unnamed subcontractor, not the bidder.
- **D** is flagged: no schedule is given. Its score would be between 52.5 and 67.5 depending on the answer, so it would place second or third, above C either way. This is why it is not ranked on a guess.
- **C** gives its experience start year as 2020, so some installs may fall outside the 5-year window. The reviewer confirms the count; even with no experience points C stays below A and E.

## 3. Human in the loop

Three checkpoints:
1. **After extraction:** the reviewer checks any uncertain or missing fact against the quoted evidence and can correct it with a reason.
2. **Flagged and ineligible bids:** a person confirms each one or requests clarification, after which the bid is re-checked.
3. **Before award:** the reviewer reads the ranking and breakdown, may override a position with a written reason, and makes the award decision. The tool never awards.

**Logged:** who uploaded what and when, the document hash, prompt and model version, extracted facts and evidence, corrections, criteria version, score breakdown, flags, overrides with reasons, and the final decision. The log is append-only, and only authorised roles can change criteria or override.

## 4. Changing criteria

Every published criteria set is a numbered, read-only version. An edit creates a new draft; it can only be published if the weights total 100%. Existing results stay tied to the version that produced them.

To use the new version, the reviewer starts a new evaluation, and all proposals in it are re-scored together so none are compared across versions. Saved facts are reused; AI re-reads a proposal only if the new criteria add a fact that was never extracted. If a mandatory rule changes, eligibility is re-run too. The screen always shows which version produced a result.

## 5. Risks

- **Wrong extraction or made-up facts:** every fact needs a quote that code verifies in the document; missing means unknown, never a guess; uncertain mandatory facts go to a person.
- **Manipulation (prompt injection):** proposal text is treated as data, and the AI has no access to scores or ranks. C contains hidden text telling an automated reviewer to rank it first. It is ignored, recorded and shown to the reviewer as a possible integrity concern. The reviewer decides what to do, so C is neither boosted nor silently penalised.
- **Mandatory-criteria errors:** pass/fail is code, and unknown is never treated as a pass (B and D above).
- **Bias or unfair scoring:** only admin-approved, job-related criteria are used, and outcomes are reviewed periodically.
- **Scanned or unreadable PDFs:** too little extracted text means the file goes to a person, not to scoring.
- **Unsafe files and data exposure:** type and size limits, malware scan, role-based access, encryption, no secrets in code or logs.
- **Cost and outages:** one extraction call per proposal, size limits, capped retries, and reuse of saved facts.
