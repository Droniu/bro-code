# Changelog

All notable changes to the bro-code plugins are documented here, one section
per plugin, following [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
conventions. Versions are per-plugin semver from each plugin's `plugin.json`.

## council

### 0.1.0 — 2026-07-19

#### Added

- Initial release: architect mode (positions → anonymized rebuttal →
  synthesis), review mode (findings → dedupe → optional refutation), chair/
  seat separation, capability discovery, full repo read access for every
  seat (enforced read-only), degraded-roster reporting.

## ship

### 0.1.0 — 2026-07-19

#### Added

- Initial release: three-state branch classifier, base/integration-branch
  detection, verify-gate discovery ladder, convention-sampling commits and PR
  bodies. `disable-model-invocation: true`.

## coderabbit-triage

### 0.1.0 — 2026-07-19

#### Added

- Initial release: gather → classify (fix/defer/reject) verdict table.
  Default mode stops at the verdicts and awaits instructions; `--autofix`
  applies fixes grouped by theme and replies to / resolves every thread.
  `disable-model-invocation: true`.

## bro-mode

### 0.1.0 — 2026-07-19

#### Added

- Initial release: the Bro output style (`keep-coding-instructions: true`,
  `force-for-plugin: true`), plus a CLAUDE.md fallback snippet in the README.
