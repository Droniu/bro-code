# Changelog

All notable changes to the bro-code plugins are documented here, one section
per plugin, following [Keep a Changelog](https://keepachangelog.com/en/1.1.0/)
conventions. Versions are per-plugin semver from each plugin's `plugin.json`.

## council

### 0.2.0 — 2026-09-12

#### Added

- Per-seat preflight: a one-turn smoke call decides the roster, so a broken
  codex install or an exhausted grok quota is caught before three seats get
  announced and two get retracted mid-run.
- Nickname resolution for seat selection ("just fable and astra"), with an
  alias table mapping nicknames to provider and model.
- Seat budgets: turn caps, a codex wall-clock timeout, and a watchdog on the
  Claude seat, so one runaway seat cannot swallow a run.
- Wall-clock expectations up front, plus a spend line drawn from the provider
  envelopes.
- An evidence gate: a seat whose answer carries no `file:line` citation is
  reported degraded instead of counted as a position.
- A pointer to `consult` for the single-model case.

#### Fixed

- Codex seats no longer use `codex exec review`, which rejects `-C`, refuses
  `--base` alongside a prompt, and ignores `--output-schema` (it returns
  prose). The diff target moved into the brief.
- `find-schema.json` now lists every property in `required`, as OpenAI strict
  structured output demands; the old schema was rejected outright.
- Grok failures are diagnosed as quota (`429`, `free-usage-exhausted`,
  `reauthable: false`) instead of advising a pointless `grok login`, and grok
  is always invoked by name so a shell wrapper's API key and `GROK_HOME`
  still apply.
- Dropped `--permission-mode plan` from the Grok seat: the flag only applies
  `default` and `bypassPermissions`, so `plan` was accepted and silently
  ignored. `--sandbox read-only` was and remains the actual enforcement.
- Corrected the Grok model and effort claims: `grok models` lists the catalog
  and `-m` pins a seat model (new `--grok-model` flag), and
  `--reasoning-effort` is silently ignored when the model's catalog entry says
  `supports_reasoning_effort: false`, so the roster now reports "effort: n/a"
  instead of a level that was never applied.
- Grok auth detection no longer probes a hardcoded `~/.grok/auth.json`, which
  missed wrapper setups that relocate `GROK_HOME`.
- The Claude seat's brief is passed as literal text and asserted non-empty,
  since workflow args can arrive stringified or empty; seats also write results
  to disk so notification truncation is off the critical path.
- Documented that a headless Grok seat ends its whole run
  (`stopReason: "Cancelled"`) at the first command needing approval.

### 0.1.0 — 2026-07-19

#### Added

- Initial release: architect mode (positions → anonymized rebuttal →
  synthesis), review mode (findings → dedupe → optional refutation), chair/
  seat separation, capability discovery, full repo read access for every
  seat (enforced read-only), degraded-roster reporting.

## consult

### 0.1.0 — 2026-09-10

#### Added

- Initial release: one-on-one consultation with a single named model —
  Codex, Grok, or a Claude subagent. The consultant's access mirrors the
  parent session's permission mode, it gets a self-contained brief, changes
  it makes to the working tree are reported, and follow-ups resume the same
  consultant session.

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
