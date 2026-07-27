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