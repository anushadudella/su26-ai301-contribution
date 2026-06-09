# AI301 Contribution README

## Status
Phase I Complete

## Issue
https://github.com/saleor/saleor/issues/15138

## Why I Chose This Issue
Saleor's sync webhooks are sometimes called inside Django database transactions, which is incorrect behavior that can cause subtle bugs. The fix requires adding a generic `logger.error` condition that fires whenever a sync webhook is called within a transaction. This makes it easy to detect all such cases across the codebase. I chose it because it's a well-scoped Python/Django task with clear acceptance criteria from a core maintainer.

## Phase II
Coming soon.

## Phase III
Coming soon.

## Phase IV
Coming soon.
