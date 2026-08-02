## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/152

**Issue title:** Faithfulness checker can never mark short claims as supported

**Tier:** [✅] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The current faithfulness checker used to test the RAG's feedback system for a user according to their current profile can not attribute feedback to context with **fewer than 2 overlapping token match**. This means that feedbacks that don't have at least 2 overlap tokens with the user's profile will be deemed as unfaithful. The problem lies within the `_is_supported()` method in `rag/evaluator/faithfulness_checker.py`, of which will trigger the caller method, `check()`, to automatically score the feedback faithfulness' score **0.0** for cases with fewer than 2 matches. A sucessful fix would mean that `_is_supported()` will not force `check()` to automatically assign a score, but score the feedback-context attribution per the contents of them, even though they might have smaller overlaps. The score shouldn't be 0.0 all the time if there are less than 2 overlaps of tokens between the RAG's feedback, and the parsed student profile. This fix is **localized and only used for tests** in `tests/unit/test_faithfulness_checker.py`, so the fix should be easy to manage.

**Scope-fit reasoning:** The scope of this issue should be **minimal**, since `faithfulness_checker.py` is only used during **unit tests, not in production**, as well as there are no external dependencies imports. All changes needed to fix this issue should be resolved in only `faithfulness_checker.py`. **This is my first time contributing to OSS** and this simple Tier 1 issue serves as the perfect stepping stone for me. 

**Branch name:** fix/152-rag-faithfulness-short-claim

**Setup confirmation:** [✅] App runs locally at localhost:5173

**Cohort ledger:** [✅] Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** [link to commit]() documenting the reproduced issue

**Reproduction summary:**
I utilized the code snipped provided in the Github Issues page. I observed the same output described in the page, with the exception of this string printed by `logger.info(...)` within the `FaithfulnessChecker.check()` method's definition:

> 2026-07-28 18:45:05 [info     ] faithfulness_checked           claims_count=1 score=0.0 supported_count=0

I played around with some other `context` and `feedback` inputs to see if the method still fails to produce the wanted result, and it sure did.

Code:

```python
from rag.evaluator.faithfulness_checker import FaithfulnessChecker
f = FaithfulnessChecker()
print(f.check(feedback = "This kid only knows C++", context_chunks=[{"text": "C"}, {"text": "C++"}])
```

Output:

> 2026-07-28 18:48:02 [info     ] faithfulness_checked           claims_count=1 score=0.0 supported_count=0
> 0.0

**PLAN.md link:** [link to PLAN.md](PLAN.md)

<!-- **Walkthrough video (recommended):** [link to your Loom video, ≤2 min — recommended, not graded] -->

<!-- **Blockers or open questions:**
[Anything you're still uncertain about going into Week 9, or leave blank] -->

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Implemented the full fix in `rag/evaluator/faithfulness_checker.py`, covering steps 1–6 of PLAN.md:
- Replaced the fixed `overlap >= 2` token count in `_is_supported()` with `_support_ratio()`, a proportion of meaningful overlapping tokens to the claim's own meaningful token count, so short claims can pass without lowering the bar for long ones.
- Reworked tokenization to use a regex (`[a-z0-9']+(?:[+/#]+[a-z0-9']*)*`) instead of raw `.split()`, so trailing punctuation ("python,") no longer breaks overlap matching while multi-symbol terms ("C++", "CI/CD") stay intact.
- Derived `SUPPORT_THRESHOLD = 0.35` by solving for a value that satisfies every existing test's inequality simultaneously.
- Added `_scale_ratio()` so `check()` produces a graded per-claim score instead of a binary supported/unsupported count (fixes single-claim feedback being forced to exactly 0.0 or 1.0).
- Fixed two edge-case bugs surfaced while testing: `chunk.get("text")` not handling explicit `None` values, and a divide-by-zero in `_support_ratio()` when a claim's tokens are all stop words.
- Updated `tests/unit/test_faithfulness_checker.py` to match (type hints, cleanup of comments referencing the old fixed-count behavior); all 22 unit tests pass.

**Next steps:**
Run `make check` and `make test-unit` for a final self-review pass, double check `eval_suite.py` doesn't assume the old binary 0.0/1.0 output (flagged as an open risk in PLAN.md), then open the PR for #152.

**Blockers:** Verifying if new functions run correctly in all cases.

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/567

**Branch:** `fix/152-rag-faithfulness-short-claim`

**What you built:**
Replaced `_is_supported()`'s fixed `overlap >= 2` token count with `_support_ratio()`, a proportional overlap measure so short claims aren't unfairly penalized, and had `check()` score each claim on a continuous gradient (`_scale_ratio()`) instead of a binary supported/unsupported count, so single-claim feedback isn't forced to exactly 0.0 or 1.0. Along the way, fixed tokenization to strip trailing punctuation without breaking multi-symbol terms ("C++", "CI/CD"), derived `SUPPORT_THRESHOLD = 0.35` from the existing test suite's constraints, and fixed three related edge-case bugs: `None` and non-string `"text"` values in a context chunk, and a divide-by-zero when a claim's tokens are all stop words.

**Tests added or updated:**
`tests/unit/test_faithfulness_checker.py` — updated the original 22 tests (type hints, removed comments describing the old fixed-count behavior) and added 19 new tests after a pressure-testing pass: direct coverage for `_scale_ratio()` and `_support_ratio()` (previously only exercised indirectly through `check()`), `_extract_claims()` boundary cases (10 vs. 11 character cutoff, no punctuation, empty string, the 10-claim cap), and type-mismatch handling (non-dict chunk items, non-string feedback, non-string `"text"` values). 41 tests total, all passing.

**Self-review confirmation:** [✅] make check passes  [✅] make test-unit passes

**Pre-existing failures (unrelated to #152):** `make check` fails on `main` with 182 ruff errors, all in files this branch never touches (e.g. `test_tech_detector.py`). Scoped `ruff`/`black`/`mypy` runs against just `faithfulness_checker.py` and `test_faithfulness_checker.py` pass clean. `make test-unit` fails on `main` with 53 failures across unrelated test files (`test_bias_detector.py`, `test_pii_scrubber.py`, `test_review_service.py`, etc.); this branch has 49 failures — same unrelated set, minus the 4 that used to fail in `test_faithfulness_checker.py` before this fix. Net: this branch introduces zero new failures and fixes 4 pre-existing ones.

**Draft PR feedback received from:** N/A - Feedback pending.
