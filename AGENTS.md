# Repository Schema

- File Standard: All curated lists must reside in `README.md`.
- Line Formatting: Maximum 120 characters per line for prose.
- Link Standard: Strict awesome-lint Markdown conventions.

## Cursor Cloud specific instructions

- This repo is a static "awesome list". Its only functionality is validating `README.md` with `awesome-lint`.
- Lint / test / build / run are all the same single command: `npm run test` (an alias for `npx awesome-lint README.md`).
- Expected non-obvious result: `awesome-lint` always reports two errors — `The repository should have "awesome"`
  / `"awesome-list" as a GitHub topic` (rule `remark-lint:awesome-github`). These are GitHub repo-metadata checks,
  not content problems, and cannot be fixed in the dev environment. A content-clean run means zero OTHER errors.

