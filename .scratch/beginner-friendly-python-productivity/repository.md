# Repository and contribution guide

The source repository is public at [sanjayshr/study-tracker](https://github.com/sanjayshr/study-tracker). Current deliverables are the preparation guide and [project/architecture documents](project.md). Application development follows once the Pi setup and remaining design decisions are clear.

## What belongs here

Commit explanatory source code, meaningful tests, setup documentation, dependency pins, and examples with invented data. Keep module boundaries simple enough for a first-year student to follow.

Documentation for this effort lives in `.scratch/beginner-friendly-python-productivity/`, as requested. The root README links to those files so visitors have an obvious entry point. `.scratch` is deliberately versioned; it is not a directory for secrets.

`package.json`, `package-lock.json`, and `.nvmrc` are repository-level deployment-tool metadata. `private: true` in the npm manifest prevents accidental npm package publication; it does not make the GitHub repository private. There is no frontend framework or JavaScript application scaffold.

When implementation begins, use small Python modules for configuration, button/gestures, session state/timing, SQLite, audio, dashboard generation, publishing/retries, and CLI entry points. Keep the state rules testable without GPIO or credentials. Do not create empty modules just to fill a proposed directory tree.

## What stays private

Keep local configuration, credentials, actual study records, backups, CSV exports, logs, and generated dashboard files out of Git. `.gitignore` covers common filenames but cannot recognise every secret or personal detail. It also does not remove files already committed.

Before a contribution, review the diff and staged files. If a token is ever committed, revoke it immediately; deleting the current file alone does not remove it from history. Report security concerns privately to the repository owner rather than opening a public issue containing a credential or personal database.

## Contributing as a beginner

Describe the problem and expected behaviour before changing code. Make one focused change, explain why it helps, and record the checks you actually ran. For hardware changes, include the OS release, architecture, kernel, and kit revision, but leave network details and credentials out.

If you have no Pi, simulation tests will be the contribution path once the application is built. For now, documentation fixes and clearly labelled compatibility reports are useful. Never claim a physical check passed because a mock passed.

The project has not selected a software licence yet. Public visibility lets people read the repository; it is not a substitute for an explicit reuse licence. Choose one before describing the project as licensed open-source software.

## Planning conventions when we resume

The wider project will be charted in `map.md`, with decision tickets in `issues/NN-slug.md`, once the destination and first decisions are agreed. Each ticket will have a descriptive title, `Type:`, `Status:`, and `Blocked by:` where needed. Open unclaimed tickets omit `Status:`; a claim uses `Status: claimed`, and resolution uses `Status: resolved` with an `## Answer`.

Implementation tickets later belong in `implementation/NN-slug.md`. Keep decisions and build tasks distinct. The OS-first request pauses that interview; no full-project map or application implementation is being implied by these preparation documents.
