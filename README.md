# AI301 Contribution README

## Status
Phase IV Complete

## Issue
[saleor/saleor#15138](https://github.com/saleor/saleor/issues/15138)

## Why I Chose This Issue
Saleor's sync webhooks are sometimes called inside Django database transactions, which is incorrect behavior that can cause subtle bugs. The fix requires adding a generic `logger.warning` condition that fires whenever a sync webhook is called within a transaction. This makes it easy to detect all such cases across the codebase. I chose it because it's a well-scoped Python/Django task with clear acceptance criteria from a core maintainer.

## Phase II

### Environment Setup
Cloned fork to local machine on macOS. Created working branch `fix-issue-15138` and pushed to remote. No Docker setup required to reproduce — the issue is identifiable through static code analysis and a unit test.

**Working branch:** https://github.com/anushadudella/saleor/tree/fix-issue-15138

### Steps to Reproduce
1. Clone the repo and open `saleor/webhook/transport/synchronous/transport.py`
2. Navigate to `send_webhook_request_sync` at line 195
3. Note it makes an outbound HTTP call via `_send_webhook_request_sync` with no check for whether a database transaction is currently active
4. **Expected:** A `logger.warning(...)` is emitted whenever this function is called inside an active Django transaction
5. **Actual:** No such warning exists — the call proceeds silently inside transactions, risking data inconsistency if the transaction rolls back

### Solution Plan

**Understand:** `send_webhook_request_sync` dispatches live HTTP calls to external apps. Calling it inside a Django `transaction.atomic()` block is dangerous — if the DB transaction rolls back, the external app has already acted on data that never committed. There is currently no logging guard to detect this pattern anywhere in the codebase.

**Match:** Django provides `connection.in_atomic_block` to detect active transactions at runtime. The file already has `logger` defined at line 63 and imports `from django.db import transaction` at line 9.

**Plan:**
1. Add `connection` to the existing `from django.db import` statement in `saleor/webhook/transport/synchronous/transport.py`
2. Add a `logger.warning` check with `stack_info=True` at the top of `send_webhook_request_sync`
3. Add a unit test in `saleor/webhook/transport/synchronous/tests/test_transport.py`

**Files touched:**
- `saleor/webhook/transport/synchronous/transport.py`
- `saleor/webhook/transport/synchronous/tests/test_transport.py`

**Review:** Followed Saleor's CONTRIBUTING.md — conventional commit message format, all existing tests pass.

**Evaluate:** New unit test confirms logger fires when called inside a transaction. All 7 tests passed.

## Phase III

### Implementation Notes
Added `connection.in_atomic_block` check inside `send_webhook_request_sync` in `saleor/webhook/transport/synchronous/transport.py`. Added `connection` to the existing `from django.db import` statement. Initially used `logger.error`, then updated to `logger.warning` with `stack_info=True` per maintainer feedback.

### Testing Strategy
Added 1 new unit test `test_send_webhook_request_sync_logs_error_inside_transaction` in `test_transport.py`. All 7 tests passed.

**Test output:**
================== 7 passed, 12 warnings in 279.96s (0:04:39) ==================

### Code Changes
https://github.com/anushadudella/saleor/tree/fix-issue-15138

## Phase IV

### Pull Request
**PR Link:** https://github.com/saleor/saleor/pull/19390
**Status:** Closed (duplicate of existing contribution #19354)
**Summary:** Added `connection.in_atomic_block` check in `send_webhook_request_sync` to log a warning when sync webhooks are called inside a DB transaction. Includes 1 new unit test confirming the logger fires when called inside a transaction.

### Before/After Evidence

**Before fix:** 6 tests passing, no transaction guard test existed, no warning emitted when sync webhook called inside transaction.

**After fix — test output:**
================== 7 passed, 12 warnings in 279.96s (0:04:39) ==================

**New test added:** `test_send_webhook_request_sync_logs_error_inside_transaction` — confirms `logger.warning` fires when `send_webhook_request_sync` is called inside `transaction.atomic()`

### Maintainer Feedback Log
| Date | Feedback | Response | Commits |
|------|----------|----------|---------|
| July 1, 2026 | Reviewer (wcislo-saleor) requested changing `logger.error` to `logger.warning` with `stack_info=True` to match async transport pattern and avoid triggering error-level incidents | Updated implementation and test accordingly | 4c4e407, 709068f |
| July 1, 2026 | PR closed as duplicate of #19354 which covers the same fix | Documented outcome in README | — |

### Outcome
PR received maintainer review, feedback was addressed (`logger.error` → `logger.warning` with `stack_info=True`), and PR was closed as a duplicate of an existing contribution. Implementation was correct and aligned with what the maintainer wanted.

## Learnings & Reflections

**Technical gains:**
- Learned how Django's `connection.in_atomic_block` works to detect active transactions at runtime
- Gained experience navigating a large real-world Django/Python codebase (Saleor has 22,000+ commits)
- Learned how to set up a full Docker-based local development environment from scratch
- Understood the difference between `logger.error` and `logger.warning` and when to use `stack_info=True`

**What I'd do differently:**
- Check for existing open PRs on an issue before starting Phase III — another contributor had a draft PR (#19354) that I didn't notice until after I opened mine
- Comment on the issue earlier to signal intent and check for duplicates
- Include test output/console logs in the PR description from the start

**Takeaway for future contributors:**
Before opening a PR on a long-open issue, always search for open AND closed PRs referencing that issue number. Maintainers on large projects often have a preferred contributor already in mind.
