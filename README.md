# coorb QA Assessment – Ahmed Gabr

## What's where

| Folder | Task | Main file |
|---|---|---|
| `task-1/` | Test design & risk coverage (Local Fund Transfer) | `task-1-test-design.md` |
| `task-2/` | Exploratory testing & bug reports (ParaBank) | `task-2-bug-reports.md`, evidence in `evidence/` |
| `task-3/part-a/` | Postman collection for the FX rates API | Collection, environment, data file, note and run output |
| `task-3/part-b/` | Testing the payments endpoint (written) | `task-3-part-b-payments-endpoint.md` |
| `task-4/` | Automation design (bank operations console) | `task-4-automation-design.md` |


## Time spent

| Task | Time spent |
|---|---|
| Task 1 | 90 min |
| Task 2 | 40 min |
| Task 3 | 60 min |
| Task 4 | 45 min |


## Assumptions, and the questions I'd have asked

**Task 1:** the full list of 14 questions, each with the assumption I tested against, is in `task-1/` (section 1.1). The six that block test design are:
- Does "local" include other banks?
- What does the customer see when we don't know whether a transfer went through?
- How is the daily limit counted?
- What are the OTP rules?
- What is the fee model?
- What are the failure reasons, and how are reversals handled?

**Task 2:**
- *Where my expectations come from:* the app's own text (for example, "A minimum of $100.00 must be deposited"), its own numbers (Available $0.00), its other screens (Account Activity versus Find Transactions), and basic banking rules.
- *Question for the PO:* is opening an account from an overdrawn account ever allowed, for example with an agreed overdraft? I assumed it isn't.

**Task 3:**

- *Questions for the PO:*
  - How old can an indicative rate be?
  - Which currencies must be supported?
  - What happens if the rates provider is down: is there a fallback provider?
  - For the payments endpoint: the real limits, how long the Idempotency-Key is kept, and the missing 401, 403 and 404 responses.

**Task 4:**
- *Assumptions:*
  - Settings can be backed up and restored through an API.
  - The settings are shared, so tests that change them run one at a time.
  - Each setting's wrapper has a `data-testid`.
- *Question:* can developers add a `data-testid` to every setting wrapper that's missing one?



## What I deliberately left out, and why

| Task | Left out | Why |
|---|---|---|
| 1 | Every fee amount, load testing, full security testing, layout, accessibility and devices | The 12-case cap. These belong to developer unit tests, separate load and security tests, and design reviews. See section 1.4. |
| 2 | Transfer Funds, Bill Pay, Request Loan, Update Contact Info | The shared demo database was reset during my session. My user was wiped and every page returned "An internal error has occurred" (see `task-2/evidence/Env_site_error_after_demo_data_reset.jpg`). |
| 3 | Repeating the negative cases for every currency | They don't depend on the data file, so they run on the first iteration only. |
