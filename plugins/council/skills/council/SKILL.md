---
name: council
description: Convene a multi-model council for adversarial code review or architecture brainstorming. Fans a question or diff out to every available model CLI (Codex, Grok) plus a sandboxed Claude seat in parallel, runs an anonymized rebuttal or refutation round, and synthesizes with disagreements surfaced. Use when the user types /council, or asks for a second opinion, multi-model review, architecture critique, or "what would other models say".
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

Flags: `--only codex,claude` select seats — the chair always remains; omitting `claude` from the list drops Claude's seat too · `--model <m>` override the Codex model · `--claude-model <opus|sonnet|haiku>` override the Claude seat model (default `opus`) · `--effort <low|medium|high|xhigh|max>` override fan-out effort (clamped per provider).

When the Claude seat runs the **same model as the chair**, say so in the roster — a sibling instance holds no stake in the outcome, but the chair reasons the way its sibling argues; weigh cross-family agreement (codex+grok) a notch higher that run.

## Prerequisites, cost, and data exposure

- **Provider CLIs are all optional.** The council runs with whoever is present: [Codex CLI](https://github.com/openai/codex) (`codex`), [Grok CLI](https://docs.x.ai) (`grok`). Each must be installed and authenticated by you, on your account, at your cost. The Claude seat needs no extra install.
- **A council of one is a degraded council.** With only the Claude seat available, the chair and the sole seat are the same model family. Say so plainly in the roster — the user is getting one family's opinion, not cross-model disagreement, which is the entire value proposition.
- **Data exposure:** every seat with repository access reads your code and sends what it reads to that provider's API. Read-only sandboxing stops writes to your disk; it is **not** a privacy control. Do not run repo-access seats on codebases you may not share with those vendors.
- **Cost:** every consultation is a real session against each provider's quota. See the sessions table at the bottom. Announce before an expensive fan-out.

## Models and reasoning effort

**Always state the roster with model and effort before running**, so the user knows what they paid for:
`Council: codex <model> @xhigh · grok <model> @high · claude opus @xhigh — chaired by claude (no seat)`

Discover capabilities instead of assuming them:

- **Codex model**: read the `model` key from `~/.codex/config.toml` if the file exists; otherwise omit `-m` and let the CLI use its default. Report whichever applies in the roster.
- **Grok model**: the CLI offers no model flag on all accounts; confirm the served model post-hoc from `.modelUsage` in the response envelope and report it.
- **Claude seat**: default `opus`, overridden only by `--claude-model`. **Never let the seat inherit the session model.** Workflow's `agent()` defaults to the main-loop model when `model:` is omitted — pass `model:` explicitly on every seat call, both modes. "The user is running X, so the seat should be X" is exactly the drift this rule exists to stop.

**Effort is set per stage, not per mode**: the fan-out (where quality is decided) runs one notch above rebuttal/refutation (many small judgment calls; diminishing returns). Known-accepted ranges as of the verified CLI versions below: codex `low…xhigh`, grok `low|medium|high`, claude `low…max`. If a CLI rejects an effort value, step down one notch and note the clamp in the roster. **Never leave effort unset** — unset effort means unpredictable cost and non-comparable answers.

| Stage | Codex | Grok | Claude |
|---|---|---|---|
| Fan-out (positions / findings) | `xhigh` | `high` | `xhigh` |
| Rebuttal / refutation | `high` | `medium` | `high` |

Ceiling asymmetry is real (claude reaches `max`, codex `xhigh`, grok `high`): when providers disagree, some of that gap is intensity rather than judgment. Say so instead of scoring it as pure signal.

## Step 1 — Detect providers

Always run this first. The council runs with whoever is present; never fail because someone is missing.

```bash
SP="<session scratchpad dir>/council"; mkdir -p "$SP"
echo "codex:  $(command -v codex >/dev/null && echo yes || echo no) auth=$(python3 -c "import json;print(json.load(open('$HOME/.codex/auth.json'))['auth_mode'])" 2>/dev/null || echo none)"
echo "grok:   $(command -v grok  >/dev/null && echo yes || echo no) auth=$([ -f "$HOME/.grok/auth.json" ] && echo yes || echo no)"
```

Claude always chairs. Its seat runs unless `--only` excludes `claude`. Report the roster with model and effort before doing work.

## Step 2 — Sandboxing (non-negotiable)

Every seat with repository access runs **read-only**: reads everything, writes nothing.

| Provider | Enforcement |
|---|---|
| **codex** | `-s read-only` |
| **grok** | `--sandbox read-only` (dedicated filesystem/network sandbox profile) plus `--permission-mode plan` |
| anything without a sandbox flag | brief-only: context inlined into the prompt, no repo access |

`--permission-mode plan` on Grok is a **permission gate, not a sandbox** — on its own it does not enforce read-only filesystem access. The `--sandbox read-only` profile is the enforcement; plan mode just suppresses write-tool approval churn on top of it. Never describe plan mode alone as sandboxing.

A read-only sandbox stops the model *writing to your disk*. It does **not** stop a vendor transmitting what it reads — see the data-exposure note above. Different threat models; never conflate them.

## Parsing provider output — the shapes differ

Envelope shapes verified 2026-07-19 against codex-cli 0.144.2 and grok 0.2.103. These CLIs version independently of this skill — treat envelope drift as an expected failure mode, not a surprise.

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
- For repo-access providers: *"Explore the repository before answering."*
- For brief-only providers: the code/context they need, inlined
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
# grok — repo access, read-only sandbox. Real object lands at .structuredOutput
grok -p "$(cat "$SP/brief.md")" --cwd "$REPO" \
  --sandbox read-only --permission-mode plan \
  --reasoning-effort "${GROK_EFFORT:-high}" \
  --output-format json --json-schema "$(cat "$SP/arch-schema.json")" \
  --max-turns 15 > "$SP/grok.json"
```

```text
Claude's seat — spawn via the Workflow tool when available: agent() declares both
model and effort, and its schema option returns a validated object:

  agent(<contents of $SP/brief.md>, {model: 'opus', effort: 'xhigh', schema: <arch-schema>})

Inline the brief text into the script (or guard args: workflow args can arrive
JSON-stringified — use `typeof args === 'string' ? args : args.brief`).

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
  },"required":["file","severity","claim","failure_scenario","confidence"],
  "additionalProperties":false}}},
"required":["findings"],"additionalProperties":false}
```

`failure_scenario` (concrete inputs → wrong output) is mandatory because it is the field a refuter can actually attack. "This is fragile" cannot be refuted; "if `items` is empty this throws" can be checked in ten seconds.

### 3b. Fan out — each in its native idiom

```bash
# codex — native review command (-m IS supported on the review subcommand)
codex exec review --uncommitted -C "$REPO" $CODEX_MODEL_ARGS \
  -c model_reasoning_effort="${EFFORT:-xhigh}" \
  --output-schema "$SP/find-schema.json" -o "$SP/codex-find.json" \
  < /dev/null "Report findings matching the schema. failure_scenario must be concrete."
```

Use `--base <branch>` or `--commit <sha>` instead of `--uncommitted` when the user specified a target.

```bash
# grok — reads the diff itself, read-only sandbox
grok -p "Review the uncommitted changes in this repository. Report findings matching the schema. failure_scenario must be concrete." \
  --cwd "$REPO" --sandbox read-only --permission-mode plan \
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
- `--max-turns` on Grok bounds runaway exploration; Codex is bounded by effort
- Announce before an expensive fan-out: *"3 seats, repo access, ~2 min, 6 sessions. Go?"*
- Narrow with `--only` or `--effort medium` when the question doesn't warrant full depth

## Failure handling

Providers fail independently and that is fine. A dead provider is a smaller council, not a failed run.

- Missing/unauthenticated → skip, name it in the roster
- Non-JSON despite schema → parse what you can (see the defensive-parsing rule), report the provider as degraded, never fabricate its position
- Grok auth errors like `invalid_grant: Refresh token has been revoked` can be triggered by token rotation racing under parallel fan-out → tell the user to re-run `grok login`
- Timeout → report partial council; do not silently drop a member

Never invent a provider's opinion. If a provider did not answer, it has no position — say that.
