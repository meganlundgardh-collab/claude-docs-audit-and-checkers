# Part 3 — Checker Prototypes

**Overview:** Two automated checkers enforcing Part 2's style guide, run against the 50-page audited slice (Skills, Plugins, Connectors, and 9 surface pages). I built the core model first, then the second (time-permitting) because I thought the failure modes would be interesting (and they were!)

**Data pipeline:** The ground-truth page index is a union of two independent sources (`llms.txt` and `sitemap.xml`). Relying on either one alone caused false positives during the build; the checkers cross-reference both and rely on a manually verified override list where the indexes disagree. *(Note: Scraping was WebFetch-based due to sandbox outbound network constraints.)*

-----
# Module 1 - Link checker (rule 3)
**`checker/link_checker.py`** — Rule 3: *"Internal links must resolve, and anchor text must match the destination's real title."* The core module. A deterministic Python module enforcing Rule 3. It extracts internal links and validates resolution and anchor-text accuracy 

## Engineering decisions & AI-assisted iteration

- **Anchor similarity refinement:** Claude's initial prototype used Jaccard similarity, which produced an artificially high 35% mismatch rate by heavily penalizing accurate, short anchors pointing to long page titles. I diagnosed the scoring flaw and directed a switch to an Overlap Coefficient. This correctly scored these as matches and cut the noise rate in half. This was a deliberate, principled fix, not just tuning to the sample.

  *Note on remaining mismatches:* the remaining 28 flags were read by hand and mostly turned out to be a *different*, still-real limitation — no stemming, so "submission" and "submitting" don't token-match even though they're the same word. I intentionally left the remaining false positives (largely caused by a lack of stemming, where "submission" and "submitting" don't match) unpatched - fixing it would have made this run's output cleaner but wouldn't be evidence the fix generalizes, and the assignment specifically asked for real output including checker mistakes rather than a scrubbed sample.

- **Fragment resolution:** I caught that the initial regex Claude and I built was silently dropping fragment links (#section), which made up over a third of the internal links. We widened the extraction and added GitHub-style slugification so the checker evaluates fragment links directly against actual H2-H6 headings rather than falling back to whole-page titles, significantly improving accuracy.   

## Evaluating the link checker
**Defining false-positive tolerance**

While building the manual overrides for index discrepancies, an automated fetch flagged third-party/claude-desktop/models as a 404. However, a manual check in my browser confirmed the page was live — the error was likely a stale cache entry or a tool-specific quirk on that route. This direct experience provided the answer to the assignment's evaluation question: a single automated fetch returning an error is never proof a link is dead. This checker's own data-prep process produced a false positive from exactly that mistake, twice, within the same hour.

A second, closer-to-home instance of the same mistake: the checker itself classified all three `/cowork/3p/extensions` links on `cowork/guide/plugins.md` as `BROKEN`, and `run_log.md`'s hand-written commentary asserted the path was "renamed or never existed" — an inference from the target not being in either index, never actually confirmed by fetching it. It was wrong; the page resolves to full real content, just missing from both indexes. Caught during an independent review pass on 2026-09-19, not by this checker's own logic — see `run_log.md`'s "What changed in the fourth run" for the fix. Kept in, not scrubbed out, for the same reason the assignment asks for checker mistakes at all: this is the more instructive failure of the two, since it's the automated classification getting it wrong, not just a flaky manual fetch during data prep.

- **Tolerance threshold:** For a BROKEN verdict, the cost of being wrong is asymmetric, so my tolerance is near-zero — under 5% of BROKEN verdicts should turn out to be false. A false BROKEN sends people chasing ghost links; if that happens frequently, users stop trusting the output entirely. This run's own false-404 incident (below) shows how easily a single bad fetch clears that bar on its own, which is why production versions must implement retry/backoff logic (e.g., requiring two failures on different days to rule out transient outages) before surfacing a page as dead.

- **A softer signal:** ANCHOR_MISMATCH is explicitly a human-in-the-loop "go look at this" signal, so the tolerance is much higher — I'd accept roughly a third of flags being noise (this run landed at 85/262, about 32%), provided the sources of that noise are documented. (For example, reading through this run's 85 mismatches revealed they were a mix of missing stemming steps, generic anchors pointing to specific subsections, and out-of-sample fallback comparisons—not metric weaknesses).

- **Detecting degradation:** Track the false-positive rate of each bucket over time via manual spot-checking. If the true-finding rate for ANCHOR_MISMATCH drops, the site's naming conventions have likely drifted from what the similarity heuristic assumes. If BROKEN's confirmed-real rate drops, it signals a fetch behavior change (e.g., redirect policies, CDN quirks) requiring a retune.

- **Preventing staleness:** The ground-truth index (known_pages.json) is only as current as its last run against the live llms.txt/sitemap.xml. It must run on a schedule or docs deploy. Similarly, manual overrides have a shelf life and must be periodically re-verified.

## Classification scheme

Each internal link on each scraped page gets exactly one status:

| Status | Meaning |
|---|---|
| `BROKEN` | Target isn't a real page in either index or the verified-overrides fallback. Attaches a "did you mean" suggestion when the anchor text closely resembles another real page's title. |
| `UNINDEXED` | Target is a real, live page but missing from `llms.txt` — Part 1 finding #3's original pattern, generalized (3 on this run — the `/cowork/3p/extensions` links, corrected from a false `BROKEN`; see `run_log.md`). |
| `ANCHOR_MISMATCH` | Target resolves and its real title is known, but the anchor text doesn't look like that title (or, for a `#fragment` link into one of the 50 scraped pages, doesn't look like the heading of the section the fragment targets — see "Anchor-title similarity," below). |
| `UNVERIFIED` | Target resolves but no title is available to compare against (not in `llms.txt`, not one of the 50 scraped pages) — deliberately *not* counted as a pass. A checker that reports "OK" when it actually has no data would be worse than one that says "can't tell." |
| `OK` | Target resolves and anchor text matches its title. |

-----

# Module 2: Semantic drift checker (rules 1 & 5)

This covers semantic_drift_checker.py (rules 1 & 5). Evaluating whether a sentence "asserts a new capability" requires semantic judgment rather than pure regex - a link resolves or it doesn't, tokens overlap or they don't - this module uses a two-stage pipeline: mechanical extraction (Stage A) followed by an LLM evaluator (Stage B).

**Disclosure: Stage B's verdicts in this submission were produced by hand, not by a live API call.** Stage A (extraction) ran fully automated and needs no model. Stage B needs `ANTHROPIC_API_KEY`, which isn't set in this build sandbox — rather than fake a result, the script stops, and what's in `output/semantic_drift_findings.md` is that same prompt applied manually, once, exactly as `call_model()` would send it. The integration code is real and would run unattended with a key. Full explanation in `output/semantic_drift_findings.md`.

## Engineering decisions & iteration

- **Expanding the context window (Stage A):** Initially, the script passed isolated sentences to the model, which caused it to miss structural violations. I refactored Stage A to capture and pass the enclosing section heading, the section's sentence count, and the full surrounding paragraph. This added context allowed Stage B to catch a hidden Rule 5 violation: a Gov skills page duplicating a 25-sentence canonical authoring guide.

- **The DISTINCT_CONCEPT category (Stage B):** I explicitly added a third classification bucket (CONSISTENT, DRIFT, DISTINCT_CONCEPT) after stress-testing against Claude Tag. Claude Tag deliberately uses the admin-scoped term "connection" differently from a personal "connector." A naive two-bucket checker would have falsely penalized this valid taxonomy difference as drift.

- **Regex widening (Stage A):** The initial extraction regex required an article ("a/an"), which missed modifier-led definitions. Widening the regex—and patching a Markdown trailing-backslash bug that was silently fusing sentences together—surfaced previously hidden candidates, including the correctly-handled Claude Tag edge cases.

- **A DRIFT verdict's direction isn't automatically "canonical is right" (caught 2026-09-19, after submission):** Two of this run's DRIFT verdicts flagged `government/desktop/plugins.md` for mentioning "hooks," a component canonical `plugins/overview.md` doesn't list. The write-up initially treated that as Government being wrong. A live cross-check of `cowork/guide/plugins.md` — a third, independent surface page — found it also lists Hooks as a real component, which is better evidence canonical is the stale page than that Government invented one. The mechanical Rule 1 verdict (local text disagrees with canonical) is still correct and still worth flagging; only the narrative about which side to fix was wrong. See `output/semantic_drift_findings.md`, "what changed in the fifth pass," for the full correction — left in rather than quietly rewritten, same principle as the link checker's false-BROKEN case above.

## Evaluating the semantic checker

- **Defining false-positive tolerance:** A false DRIFT verdict is high-cost; sending a docs team to edit content that was never wrong burns trust capital quickly. My tolerance is under 10% — tighter than ANCHOR_MISMATCH's, since DRIFT triggers an actual edit request rather than just a look. The concrete mechanism that keeps the rate under that bar isn't a confidence threshold on the model's output — it's the DISTINCT_CONCEPT bucket, which ensures the checker isn't forced to mislabel valid domain distinctions as drift in the first place.

- **Detecting degradation:** Two mechanisms are needed. First, track Stage A's candidate extraction counts run-over-run; an unexplained drop indicates the regex is silently missing new edge cases. Second, the 11 definitional candidates extracted in this run serve as a free, hand-labeled golden set. Re-running Stage B against them after an LLM version update provides a direct regression check.

- **Preventing staleness:** The LLM evaluator cannot rely on hardcoded model strings (e.g., claude-sonnet-4-5, which is scheduled for deprecation). I moved this to an environment variable (ANTHROPIC_MODEL) and wrapped the API call to catch failures with clear error logs pointing to the model ID — turning a silent crash from a deprecated model into a loud, fixable configuration update. Additionally, the hardcoded list of surface pages must be dynamically generated in production so newly shipped pages aren't silently skipped.

## Files

- `checker/build_page_index.py` — builds `data/known_pages.json` from `sitemap-urls.txt` + `llms-txt-raw.txt`.
- `checker/link_checker.py` — Rule 3 checker; run this for link validity.
- `checker/semantic_drift_checker.py` — Rule 1 checker; run this for definition drift. See "Second module," above.
- `data/verified_overrides.json` — hand-verified live status for the 6 pages the two indexes disagree on (or, for `/cowork/3p/extensions`, both indexes miss), including the `models` false-404 incident described above and the checker's own false-BROKEN verdict on `/cowork/3p/extensions` (see `run_log.md`).
- `scrape/` — 50 verbatim page snapshots (WebFetch, literal-content prompts; see the environment-constraint note above).
- `output/findings.json` — structured output of the last `link_checker.py` run.
- `output/run_log.md` — hand-annotated read-through of that run's actual findings, including which ANCHOR_MISMATCH flags look like real problems vs. checker artifacts.
- `output/semantic_drift_candidates.json` — Stage A output of `semantic_drift_checker.py`: extracted local definitions and their exact Stage B prompts.
- `output/semantic_drift_findings.md` — Stage B verdicts for this run, and how they were produced.
