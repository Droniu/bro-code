---
name: council
description: Convene a multi-model council for adversarial code review or architecture brainstorming. Fans a question or diff out to every available model CLI (Codex, Grok) plus a sandboxed Claude seat in parallel, runs an anonymized rebuttal or refutation round, and synthesizes with disagreements surfaced. Use when the user types /council, or asks for a multi-model review, cross-model disagreement, or an architecture critique from several models at once. For one named model, use consult instead.
---

# Council

Claude Code chairs. Every model — including Claude — argues from a seat with a declared model and effort; the chair holds no position of its own and validates the outcome against the source. Disagreement between seats is the deliverable, not a problem to average away.

## Usage

```text
/council architect "<question>"      positions → rebuttal → synthesis
/council architect "<q>" --solo      skip rebuttal round (faster, cheaper)
/council review                      review current diff, all providers
/council review --deep               + cross-model refutation of findings
/council review --base main          review against a base branch
```

Flags: `--only codex,claude` select seats — the chair always remains; omitting `claude` from the list drops Claude's seat too · `--model <m>` override the Codex model · `--claude-model <opus|sonnet|haiku>` override the Claude seat model (default `opus`) · `--grok-model <slug>` pin the Grok seat model (default: the CLI's own default) · `--effort <low|medium|high|xhigh|max>` override fan-out effort (clamped per provider).

When the Claude seat runs the **same model as the chair**, say so in the roster — a sibling instance holds no stake in the outcome, but the chair reasons the way its sibling argues; weigh cross-family agreement (codex+grok) a notch higher that run.

**Users name seats by model nickname**, not by provider — "drop grok, just fable and astra". Resolve nicknames before anything else:

| The user says | Seat | Resolve against |
|---|---|---|
| `opus`, `sonnet`, `haiku`, `fable` | claude | the `--claude-model` value |
| `astra`, `sol`, `terra`, `luna`, `gpt-*`, `codex` | codex | `.models[].slug` in `~/.codex/models_cache.json` (`gpt-6 astra` → `gpt-6-astra`) |
| `grok*` | grok | `grok models` |

A nickname you cannot resolve is a question for the user, never a silent substitution into somebody else's default.

**One model is not a council — that is `/consult`.** A single named model with your session's access and no rebuttal round is the lighter tool; reach for the council when you want cross-family disagreement.

## Prerequisites, cost, and data exposure

- **Provider CLIs are all optional.** The council runs with whoever is present: [Codex CLI](https://github.com/openai/codex) (`codex`), [Grok CLI](https://docs.x.ai) (`grok`). Each must be installed and authenticated by you, on your account, at your cost. The Claude seat needs no extra install.
- **A council of one is a degraded council.** With only the Claude seat available, the chair and the sole seat are the same model family. Say so plainly in the roster — the user is getting one family's opinion, not cross-model disagreement, which is the entire value proposition.
- **Data exposure:** every seat reads your code and sends what it reads to that provider's API. Read-only sandboxing stops writes to your disk; it is **not** a privacy control. Do not convene the council on codebases you may not share with those vendors.
- **Cost:** every consultation is a real session against each provider's quota. See the sessions table at the bottom. Announce before an expensive fan-out.

## Models and reasoning effort

**Always state the roster with model and effort before running**, so the user knows what they paid for:
`Council: codex <model> @xhigh · grok <model> @high · claude opus @xhigh — chaired by claude (no seat)`

Discover capabilities instead of assuming them:

- **Codex model**: read the `model` key from `~/.codex/config.toml` if the file exists; otherwise omit `-m` and let the CLI use its default. Report whichever applies in the roster.
- **Grok model**: `grok models` prints the catalog and marks the default; pin one with `-m <slug>`. Confirm the served model post-hoc from `.modelUsage` in the response envelope and report that.
- **Claude seat**: default `opus`, overridden only by `--claude-model`. **Never let the seat inherit the session model.** Workflow's `agent()` defaults to the main-loop model when `model:` is omitted — pass `model:` explicitly on every seat call, both modes. "The user is running X, so the seat should be X" is exactly the drift this rule exists to stop.

**Effort is set per stage, not per mode**: the fan-out (where quality is decided) runs one notch above rebuttal/refutation (many small judgment calls; diminishing returns). Known-accepted ranges as of the verified CLI versions below: codex `low…ultra` (per-model `supported_reasoning_levels`), grok `none…xhigh` **only where the model supports effort at all**, claude `low…max`. Grok silently ignores `--reasoning-effort` when its catalog marks the model `supports_reasoning_effort: false` — true of API-key accounts today. Check `models_cache.json` beside the `user` layer path in `grok inspect --json` → `.configSources.layers`, and report "effort: n/a" rather than claiming a level you did not get. If a CLI rejects an effort value, step down one notch and note the clamp in the roster. **Never leave effort unset** — unset effort means unpredictable cost and non-comparable answers.

| Stage | Codex | Grok | Claude |
|---|---|---|---|
| Fan-out (positions / findings) | `xhigh` | `high` | `xhigh` |
| Rebuttal / refutation | `high` | `medium` | `high` |

Ceiling asymmetry is real (claude reaches `max`, codex `xhigh`, grok `xhigh` where effort applies at all): when providers disagree, some of that gap is intensity rather than judgment. Say so instead of scoring it as pure signal.

## Step 1 — Detect providers, then smoke-test them

`command -v` plus an auth file proves nothing: a CLI can be installed, authenticated, and still fail every call — a broken helper binary, an exhausted quota, a revoked token. **Preflight every seat with a real call, and let the answer decide the roster.** Two one-turn `low`-effort sessions buy you a roster that is actually true.

Run each block as its **own** Bash call — one long chained command trips the permission splitter and stalls the whole detection step waiting for approval.

```bash
SP="<session scratchpad dir>/council"; mkdir -p "$SP"; command -v codex; command -v grok
```

```bash
timeout 90 codex exec --ephemeral -s read-only -c model_reasoning_effort=low \
  -o "$SP/smoke-codex.txt" < /dev/null "Reply with exactly: OK"; echo "codex exit=$?"; cat "$SP/smoke-codex.txt"
```

```bash
timeout 90 grok -p "Reply with exactly: OK" --output-format json --max-turns 1 > "$SP/smoke-grok.json"; echo "grok exit=$?"
python3 -c "import json;d=json.load(open('$SP/smoke-grok.json'));print(d.get('stopReason'),'|',(d.get('text') or d.get('message') or '')[:120])"
```

A seat that does not come back `OK` **is not on the roster**. Name it and its reason in one line and convene without it. Two seats announced honestly beat three announced and two retracted mid-run.

Claude chairs and needs no smoke test. Its seat runs unless `--only` excludes `claude`.

## Step 2 — Sandboxing (non-negotiable)

Every seat has **full repository read access**, enforced read-only: reads everything, writes nothing.

| Provider | Enforcement |
|---|---|
| **codex** | `-s read-only` |
| **grok** | `--sandbox read-only` (dedicated filesystem/network sandbox profile) |

Grok's `--permission-mode` flag only applies `default` and `bypassPermissions`; `plan` is accepted and silently ignored (verified on grok 0.2.103), so it is no longer passed — it never gated anything. `--sandbox read-only` is the entire enforcement. Note also that a headless grok seat **ends its whole run** (`stopReason: "Cancelled"`) the first time it reaches for a command needing approval: report that seat as degraded rather than empty.

A read-only sandbox stops the model *writing to your disk*. It does **not** stop a vendor transmitting what it reads — see the data-exposure note above. Different threat models; never conflate them.

## Parsing provider output — the shapes differ

Envelope shapes verified 2026-09-12 against codex-cli 0.153.2 and grok 0.2.103. These CLIs version independently of this skill — treat envelope drift as an expected failure mode, not a surprise.

| Provider | Where the object actually is |
|---|---|
| **codex** | Bare JSON, written straight to the `-o` file |
| **grok** | Wrapped — real object at **`.structuredOutput`**. Siblings: `.text` (same thing, stringified), `.thought`, `.usage`, `.modelUsage` |

```bash
extract() {  # $1 = provider, $2 = file
  case "$1" in
    grok) python3 -c "import json,sys;print(json.dumps(json.load(open(sys.argv[1]))['structuredOutput']))" "$2" ;;
    *)    cat "$2" ;;
  esac
}
```

**Parse defensively.** If the expected shape is missing (schema change in a newer CLI), fall back to capturing the raw text output, report that provider as **degraded** in the synthesis, and never fabricate structure that isn't there. `.usage.total_tokens` and `.modelUsage` on Grok give real per-call cost — surface it when reporting an expensive run; `.modelUsage` also names the served model.

## Mode: architect

### 2a. Write the brief

Identical prompt to every provider. Write it to `$SP/brief.md` so all providers demonstrably get the same input.

Required contents:

- The user's question **verbatim**. Do not paraphrase — your framing is a bias vector.
- Relevant constraints from CLAUDE.md/AGENTS.md and the actual stack
- *"Explore the repository before answering."* — every seat has full repo read access
- *"Respond in a neutral technical register."*

That last line matters: if a provider's global config (`~/.codex/AGENTS.md`, grok custom rules) applies a persona, you would otherwise measure manufactured attitude and score it as genuine disagreement.

### 2b. Schema

`$SP/arch-schema.json`:

```json
{"type":"object","properties":{
  "position":{"type":"string"},
  "reasoning":{"type":"string"},
  "risks":{"type":"array","items":{"type":"string"}},
  "key_assumption":{"type":"string"},
  "confidence":{"type":"string","enum":["high","medium","low"]}
},"required":["position","reasoning","risks","key_assumption","confidence"],
"additionalProperties":false}
```

`key_assumption` is the highest-value field. Models rarely disagree on logic; they disagree because they assumed different things about scale, team size, or latency budget. Surfacing the assumption usually dissolves the disagreement or reveals it as the real question.

### 2c. Fan out — parallel, one message, multiple Bash calls

```bash
# codex — repo access. $CODEX_MODEL_ARGS is "-m <model>" when ~/.codex/config.toml
# defines one, else empty (CLI default).
codex exec --ephemeral -s read-only -C "$REPO" $CODEX_MODEL_ARGS \
  -c model_reasoning_effort="${EFFORT:-xhigh}" \
  --output-schema "$SP/arch-schema.json" -o "$SP/codex.json" \
  < /dev/null "$(cat "$SP/brief.md")"
```

```bash
# grok — repo access, read-only sandbox. $GROK_MODEL_ARGS is "-m <slug>" when a
# model is pinned, else empty. Drop --reasoning-effort when effort is n/a.
# Real object lands at .structuredOutput
grok -p "$(cat "$SP/brief.md")" --cwd "$REPO" $GROK_MODEL_ARGS \
  --sandbox read-only \
  --reasoning-effort "${GROK_EFFORT:-high}" \
  --output-format json --json-schema "$(cat "$SP/arch-schema.json")" \
  --max-turns 15 > "$SP/grok.json"
```

```text
Claude's seat — spawn via the Workflow tool when available: agent() declares both
model and effort, and its schema option returns a validated object:

  agent(<the literal text of $SP/brief.md>, {model: 'opus', effort: 'xhigh', schema: <arch-schema>})

Read the brief and inline its text when you author the script. NEVER pass it
through workflow args: they can arrive JSON-stringified or empty, and a seat
handed an empty brief burns a full run refusing to invent a position from
nothing. Assert the brief is non-empty before launching, and treat a seat that
reports an empty task as a re-launch, not as a position.

Tell the seat to write its answer to $SP/claude.json as well as returning it.
Task notifications truncate long results, and digging the full object back out
of the workflow journal costs a round trip you do not need.

Fallback when Workflow is unavailable: the Agent tool. It has NO effort
parameter — flag the seat as "effort: session default (uncontrolled)" in the
roster so the asymmetry is on the record.
```

The user invoking `/council` is the explicit opt-in that Workflow orchestration requires. `model: 'opus'` is literal, not a placeholder for the session model.

**You do not hold a position yourself.** You are the chair. Claude's argument comes from the subagent, on the record with a declared model and effort exactly like the other seats. If you both argue a position and judge the outcome, you are scoring your own case — and synthesis is precisely where a thumb on the scale does the most damage.

### 2d. Rebuttal round (skip with `--solo`)

Give every provider all *other* positions, anonymized as "Model A / Model B" — labels invite deference to brand rather than argument. Include the provider's own prior position for reference (a fresh session does not remember it).

Ask for exactly: which position is strongest and why · which is weakest and why · what did everyone miss · does seeing these change your own position (yes/no + what).

Rebuttal uses the **rebuttal-row** efforts and this schema (`$SP/rebuttal-schema.json`):

```json
{"type":"object","properties":{
  "strongest":{"type":"string"},
  "weakest":{"type":"string"},
  "missed":{"type":"string"},
  "position_changed":{"type":"boolean"},
  "revised_position":{"type":"string"}
},"required":["strongest","weakest","missed","position_changed","revised_position"],
"additionalProperties":false}
```

Every field is `required` because strict structured output rejects optional properties — `revised_position` is `""` when `position_changed` is false.

Position changes are the single most informative output. A model that revises under argument was reasoning; one that never budges was pattern-matching.

### 2e. Synthesize

```text
CONSENSUS n/N — <the agreed position>
CONTESTED — <topic>
  codex   <position>  (assumed: <key_assumption>)
  grok    <position>  (assumed: <key_assumption>)
  claude  <position>  (assumed: <key_assumption>)
  → resolves to a disagreement about <the actual root question>
CHANGED POSITION — <who moved, after what argument>
MY CALL — <your judgment, with the failure case that decides it>
```

You are the validation layer and the chair, not a vote counter and not a competitor. If every member agrees and they are all wrong, say so — with evidence from the code. Majority is not correctness. When seats disagree on a checkable fact, **check it yourself** before synthesizing — the chair's authority comes from having read the source and holding no position it needs to win.

## Mode: review

### 3a. Schema

`$SP/find-schema.json`:

```json
{"type":"object","properties":{"findings":{"type":"array","items":{
  "type":"object","properties":{
    "file":{"type":"string"},
    "line":{"type":"integer"},
    "severity":{"type":"string","enum":["critical","high","medium","low"]},
    "category":{"type":"string"},
    "claim":{"type":"string"},
    "failure_scenario":{"type":"string"},
    "confidence":{"type":"string","enum":["high","medium","low"]}
  },"required":["file","line","severity","category","claim","failure_scenario","confidence"],
  "additionalProperties":false}}},
"required":["findings"],"additionalProperties":false}
```

`failure_scenario` (concrete inputs → wrong output) is mandatory because it is the field a refuter can actually attack. "This is fragile" cannot be refuted; "if `items` is empty this throws" can be checked in ten seconds.

### 3b. Fan out — each in its native idiom

```bash
# codex — plain `exec`, never `exec review`: that subcommand rejects -C, refuses
# --base alongside a prompt, and ignores --output-schema (it answers in prose).
codex exec --ephemeral -s read-only -C "$REPO" $CODEX_MODEL_ARGS \
  -c model_reasoning_effort="${EFFORT:-xhigh}" \
  --output-schema "$SP/find-schema.json" -o "$SP/codex-find.json" \
  < "$SP/review-brief.md"
```

The diff target belongs in the brief, not in flags: *"Review the uncommitted changes (`git diff HEAD`)"*, or `git diff main...HEAD` when the user named a base. Append the schema to the brief too — a seat that loses the flag still answers in shape.

```bash
# grok — reads the diff itself, read-only sandbox
grok -p "Review the uncommitted changes in this repository. Report findings matching the schema. failure_scenario must be concrete." \
  --cwd "$REPO" $GROK_MODEL_ARGS --sandbox read-only \
  --reasoning-effort "${GROK_EFFORT:-high}" \
  --output-format json --json-schema "$(cat "$SP/find-schema.json")" \
  --max-turns 20 > "$SP/grok-find.json"
```

```text
Claude's seat — Workflow agent(), same rules as architect mode:
  agent("Review the uncommitted diff in <REPO>. Findings return only via the
         structured output.",
        {model: 'opus', effort: 'xhigh', schema: <find-schema>})
```

Same rule: you chair, you do not file findings of your own. And same rule on the seat: `model:` and `effort:` explicit, `opus` default, never the session model.

### 3c. Normalize and dedupe

Merge on `file` + `line` proximity (±3) + semantic overlap of `claim`. Track which providers reported each finding — **independent co-discovery by different model families is the strongest signal available.** A bug found by two different families separately is far more likely real than one found twice by variants of the same family.

### 3d. Refutation round (`--deep`)

Not a debate. Each finding goes to a **different** provider than the one that raised it:

> "Here is a claimed defect: `<claim>` / `<failure_scenario>` at `<file>:<line>`. Read the code and try to REFUTE it. Is the scenario actually reachable? Does existing validation prevent it? Default to `refuted: true` when uncertain."

Defaulting to refuted is deliberate — AI review's dominant failure mode is confident false positives, so the burden of proof sits with the finding.

**Bound the cost.** Refute only `critical` and `high`. State what was skipped:
`Refuted 6 of 14 findings (critical+high only); 8 medium/low reported unverified.`
Never silently truncate — silent caps read as "we checked everything."

### 3e. Report

Rank: co-discovered + survived refutation → single-source + survived → refuted (listed separately, briefly).

You have the final say. If a finding survived every model but you can read the code and see it is wrong, kill it and explain. Three models agreeing is not evidence; the code is evidence.

**Keep the process out of the deliverable.** The roster, the degraded seats, and the retry story belong in the chat with the user — never in a PR description, commit message, or review comment. When the user does want provenance in a shipped artifact, name the models and nothing else.

## Cost

Every consultation is a real session doing real work against each provider's quota.

Full roster is 3 seats — codex, grok, claude. Sessions per run:

| Command | Sessions |
|---|---|
| `architect --solo` | 3 |
| `architect` | 6 (3 fan-out + 3 rebuttal) |
| `review` | 3 |
| `review --deep` | 3 + one per critical/high finding |

- `--deep` is the expensive path — never make it the default
- **Cap every seat.** Grok takes `--max-turns`; wrap codex in `timeout 900`; give the Claude seat an explicit turn budget in its brief plus a wall-clock watchdog. An uncapped seat can run for hours and return nothing, turning a three-seat council into one.
- **Say how long it will take.** A seat with repo access runs for minutes, not seconds, and a full architect run with the rebuttal round can take half an hour. Quote that up front — half an hour is a different product from "a couple of minutes".
- Announce before an expensive fan-out: *"3 seats, repo access, ~30 min, 6 sessions. Go?"*
- **Report what it cost.** Grok's envelope carries `total_cost_usd` and `usage`, codex reports tokens. Add one spend line to the synthesis for a metered run, and write "cost not reported" rather than implying it was free.
- Narrow with `--only` or `--effort medium` when the question doesn't warrant full depth

## Failure handling

Providers fail independently and that is fine. A dead provider is a smaller council, not a failed run.

- Missing/unauthenticated → skip, name it in the roster
- Non-JSON despite schema → parse what you can (see the defensive-parsing rule), report the provider as degraded, never fabricate its position
- **Grok quota, not auth.** `429`, `subscription:free-usage-exhausted`, `You've reached your free Grok Build usage limit`, or `reauthable: false` all mean the seat is out of budget. `grok login` fixes none of them — report "grok seat out of quota", give the reset window if the error carries one, and convene without it. One repo-reading seat can eat most of a free daily allowance in a single run.
- **Grok auth routing.** Invoke `grok` by name so a shell function that exports `XAI_API_KEY` or relocates `GROK_HOME` still applies; calling a resolved binary path silently drops to the free tier. `grok models` prints the route in use, and `.modelUsage` names the model actually served — `grok-4.5` served as `grok-4.5-build-free` is a degraded seat, not the model you asked for.
- Genuine auth errors (`invalid_grant: Refresh token has been revoked`) can be triggered by token rotation racing a parallel fan-out → re-run `grok login`
- **Schema-valid is not evidence of work.** A seat can return well-formed JSON without ever opening the repo. No `file:line` citation, or a single-turn finish, means it did not read your code — mark it degraded and say which.
- Timeout → report partial council; do not silently drop a member. A late seat that lands after you called it dead reopens the synthesis; never publish a verdict a background seat is still contradicting

Never invent a provider's opinion. If a provider did not answer, it has no position — say that.
