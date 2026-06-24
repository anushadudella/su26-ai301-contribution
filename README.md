# AI301 Contribution README

## Status
Phase II Complete

## Issue
https://github.com/saleor/saleor/issues/15138

## Why I Chose This Issue
Saleor's sync webhooks are sometimes called inside Django database transactions, which is incorrect behavior that can cause subtle bugs. The fix requires adding a generic `logger.error` condition that fires whenever a sync webhook is called within a transaction. This makes it easy to detect all such cases across the codebase. I chose it because it's a well-scoped Python/Django task with clear acceptance criteria from a core maintainer.

## Phase II

### Environment Setup
Cloned fork to local machine on macOS. Created working branch `fix-issue-15138` 
and pushed to remote. No Docker setup required to reproduce — the issue is 
identifiable through static code analysis and a unit test.

**Working branch:** https://github.com/anushadudella/saleor/tree/fix-issue-15138

### Steps to Reproduce
1. Clone the repo and open `saleor/webhook/transport/synchronous/transport.py`
2. Navigate to `send_webhook_request_sync` at line 195
3. Note it makes an outbound HTTP call via `_send_webhook_request_sync` with 
   no check for whether a database transaction is currently active
4. **Expected:** A `logger.error(...)` warning is emitted whenever this function 
   is called inside an active Django transaction
5. **Actual:** No such warning exists — the call proceeds silently inside 
   transactions, risking data inconsistency if the transaction rolls back

### Solution Plan

**Understand:** `send_webhook_request_sync` dispatches live HTTP calls to 
external apps. Calling it inside a Django `transaction.atomic()` block is 
dangerous — if the DB transaction rolls back, the external app has already 
acted on data that never committed. There is currently no logging guard to 
detect this pattern anywhere in the codebase.

**Match:** Django provides `connection.in_atomic_block` to detect active 
transactions at runtime. The file already has `logger` defined at line 63 
and imports `from django.db import transaction` at line 9, so no new imports 
are needed.

**Plan:**
1. Add `from django.db import connection` to the imports in 
   `saleor/webhook/transport/synchronous/transport.py`
2. Add this check at the top of `send_webhook_request_sync` (line 195):
```python
   if connection.in_atomic_block:
       logger.error(
           "Sync webhook called inside a database transaction. "
           "Event: %s. This may cause data inconsistency if the "
           "transaction is rolled back.",
           delivery.event_type,
       )
```
3. Add a unit test in 
   `saleor/webhook/transport/synchronous/tests/test_transport.py` that 
   wraps a call to `send_webhook_request_sync` inside `transaction.atomic()` 
   and asserts `logger.error` was called

**Files to touch:**
- `saleor/webhook/transport/synchronous/transport.py` (add logger check)
- `saleor/webhook/transport/synchronous/tests/test_transport.py` (add test)

**Review:** Will follow Saleor's CONTRIBUTING.md — run `ruff` for formatting, 
use conventional commit message format, ensure all existing tests pass.

**Evaluate:** New unit test should pass confirming logger.error fires inside 
a transaction. Existing test suite should show no regressions.

[## Phase III

**Implementation Notes:** Added `connection.in_atomic_block` check inside `send_webhook_request_sync` in `saleor/webhook/transport/synchronous/transport.py`. Also added `connection` to the existing `from django.db import` statement.

**Testing Strategy:** Added 1 new unit test `test_send_webhook_request_sync_logs_error_inside_transaction` in `test_transport.py`. All 7 tests pass.

**Code Changes:** https://github.com/anushadudella/saleor/tree/fix-issue-15138

## Phase IV
Coming soon.
