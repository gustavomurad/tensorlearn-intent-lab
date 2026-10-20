# Intent

## Goal
A support coordinator needs to preview the tickets in a CSV file before any later
import is approved. The future change adds a pure preview function that takes CSV
text and returns either the full list of parsed tickets, in input order, or a single
error code explaining why the file was rejected. Nothing is saved or changed: the
coordinator only sees what an import would produce.

This document describes that future work. In this round, do not implement the
feature. Explain the scope and acceptance criteria if asked, and ask the user for
explicit approval before writing any implementation code.

## Scope
Allowed changes, once the user approves implementation, are limited to two new files:

- `src/import-tickets.mjs`: exports the preview function.
- `tests/import-tickets.test.mjs`: `node:test` tests covering every acceptance case below.

Out of scope (non-goals): any UI, any database or persistence, any network calls,
file reads or writes, new package dependencies, and the actual import step itself.
If a request seems to need anything beyond the two allowed files, stop and ask the
user instead of expanding scope.

## Constraints
- Do not modify the protected paths: `src/tickets.mjs` (the stable ticket
  representation), anything under `.github/` (the grading workflow) and `CLAUDE.md`
  (Claude's instruction loader). Reuse `ticket()` from `src/tickets.mjs` rather than
  duplicating its rules where that helps.
- The preview is side-effect free (`preview-only`): it takes a CSV string and returns
  a plain object. No file system, network, environment or global state access.
- No package dependencies; use plain JavaScript ES modules on Node 22 or later.
- The result is all-or-nothing. Success is `{"tickets":[{"title":"...","priority":"normal"}]}`
  with tickets in input order. Failure is `{"error":"<code>"}`, and no partial
  ticket list is ever returned alongside or instead of an error.
- Only fictional example data appears in tests and documents.

## Plan
1. Ask the user for approval to implement before creating any file.
2. Write `tests/import-tickets.test.mjs` first, with one test per acceptance case in
   the contract below, including the 101-row case.
3. In `src/import-tickets.mjs`, parse CSV text with a small quote-aware parser that
   handles quoted fields containing commas, escaped double quotes (`""`) and both
   `\n` and `\r\n` line endings. Ignore a single trailing newline.
4. Validate the header: it must contain `title`, may contain `priority`, and must not
   contain any other, missing (blank) or duplicate column names. An empty CSV with no
   header, a bad header, or a malformed row (wrong field count or unterminated quote)
   all return `{"error":"unknown-column"}`.
5. Count data rows before validating any values; more than 100 returns
   `{"error":"row-limit"}`.
6. For each row in order: trim values; an empty title returns `empty-title`; a missing
   or blank priority becomes `normal`; a priority other than `low`, `normal` or
   `urgent` returns `invalid-priority`. Stop at the first error.
7. Return `{"tickets":[...]}` only when every row is valid, then run verification.

## Acceptance
The implementation is complete when every case in this contract produces exactly its
expected output. `repeat: 101` means the single data row is repeated 101 times.

```json
{
  "schema": 1,
  "change": "ticket-csv-preview",
  "allowed_changes": ["src/import-tickets.mjs", "tests/import-tickets.test.mjs"],
  "protected_paths": ["src/tickets.mjs", ".github/", "CLAUDE.md"],
  "non_goals": ["ui", "database", "network"],
  "requirements": {
    "columns": ["title", "priority"],
    "default_priority": "normal",
    "valid_priorities": ["low", "normal", "urgent"],
    "reject_unknown_columns": true,
    "reject_empty_title": true,
    "max_rows": 100,
    "side_effects": "preview-only"
  },
  "verification": ["npm test", "git diff --check"],
  "approval": "ask-before-implementation",
  "acceptance_cases": [
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "expected": {"tickets":[{"title":"Printer issue","priority":"urgent"}]}
    },
    {
      "csv": "title\nPassword reset\n",
      "expected": {"tickets":[{"title":"Password reset","priority":"normal"}]}
    },
    {
      "csv": "title,priority\nVPN drops,\n",
      "expected": {"tickets":[{"title":"VPN drops","priority":"normal"}]}
    },
    {
      "csv": "title,priority\n\"Question about billing, October\",low\n",
      "expected": {"tickets":[{"title":"Question about billing, October","priority":"low"}]}
    },
    {
      "csv": "title,priority\n   ,urgent\n",
      "expected": {"error":"empty-title"}
    },
    {
      "csv": "title,owner\nPrinter issue,Sam\n",
      "expected": {"error":"unknown-column"}
    },
    {
      "csv": "title,priority\nPrinter issue,critical\n",
      "expected": {"error":"invalid-priority"}
    },
    {
      "csv": "title,priority\nPrinter issue,urgent\n",
      "repeat": 101,
      "expected": {"error":"row-limit"}
    }
  ]
}
```

Positive examples: an explicit valid priority is kept; a title-only file or a blank
priority defaults to `normal`; a quoted title keeps its comma. Negative examples: a
whitespace-only title, an unknown `owner` column, the unsupported `critical` priority,
and 101 data rows each return exactly one error code and no tickets.

## Verification
After an approved implementation, run `npm test` and confirm that both the existing
ticket tests and the new import tests pass, then run `git diff --check` to confirm
there are no whitespace errors. Confirm with `git status` that only the two allowed
files were added and no protected path changed. Report the results honestly. Ask the
user for approval before implementing, before expanding scope, and before any commit,
push or PR; never commit, push or submit on your own.
