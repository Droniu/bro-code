---
name: consult
description: Consult one other model — Codex (GPT), Grok, or a Claude model — on the issue at hand, with the same access as the current session. Use when the user types /consult, or asks for one named model's take on the current problem ("ask gpt-6 about this", "what does grok think", "have sonnet look at it"). For a multi-model fan-out with rebuttal, use council instead.
argument-hint: "<model> [(codex|grok|claude)] [on <question>] [--effort <level>] [--mode <permission-mode>]"
---

# Consult

One outside model, one issue, **the same access as this session**. You stay the lead: the consultant investigates and answers, you relay that answer faithfully, check it against the code, and keep driving the task.

## Usage

```text
/consult gpt-6 astra (codex) on this issue
/consult grok on why the build broke — let it try fixing it
/consult sonnet on whether SSE or websockets fits here
```

Flags: `--effort <low|medium|high|xhigh|max>` (default: this session's effort) · `--mode <plan|default|acceptEdits|auto|dontAsk|bypassPermissions>` (default: this session's permission mode).

Follow-ups — "ask it why", "tell codex the test passes now" — go to the same consultant session (Step 5).

## Step 1 — Resolve the consultant

| Signal in the arguments | Provider |
|---|---|
| `(codex)` / `(grok)` / `(claude)` | that provider — wins over everything else |
| `gpt-*`, `o<digit>*`, `codex` | codex |
| `grok*` | grok |
| `claude`, `opus`, `sonnet`, `haiku`, `fable` | claude |
| no model or provider named | ask the user, offering installed providers only |

Turn spoken names into slugs (`gpt-6 astra` → `gpt-6-astra`) and **check the slug against the provider's own catalog**:

- **codex** — slugs in `~/.codex/models_cache.json` (`.models[].slug`, with `supported_reasoning_levels` per model). No model named → the `model` key in `~/.codex/config.toml`, else the CLI default.
- **grok** — `grok models`. No model named → its listed default.
- **claude** — the Agent tool takes `opus | sonnet | haiku | fable`. No model named → `opus`. A pinned version ("opus 4.8") can't be guaranteed through an alias — say so.

An unknown slug gets the catalog shown and a question; a CLI rejecting a model gets reported. Neither is a cue to drop `-m` and quietly run the default model instead.

`command -v codex` / `command -v grok` must succeed. Invoke them by name, never by a resolved binary path — `grok` in particular may be a shell function that sets its home and credentials. Auth failures surface on the first call — report them; never swap in a different provider.

## Step 2 — Resolve the session's access

The consultant runs in the same working directory, loads its own full user config (instructions, MCP servers, plugins, rules — Grok additionally reads Claude's `settings.json`, `CLAUDE.md`, hooks, and plugins), has network, and gets **this session's permission mode** mapped onto the provider.

**Mode** — first match wins; name the source in the roster:

1. `--mode` flag
2. The latest system reminder says plan mode is active → `plan`
3. `permissions.defaultMode` in managed settings (`/Library/Application Support/ClaudeCode/managed-settings.json` on macOS, `/etc/claude-code/managed-settings.json` on Linux)
4. A flag on this session's process: `ps -o command= -p "$CLAUDE_PID"` (`--permission-mode <m>`; `--dangerously-skip-permissions` → `bypassPermissions`)
5. `permissions.defaultMode` in `.claude/settings.local.json`, then `.claude/settings.json`, then `~/.claude/settings.json`
6. `default`

A mid-session Shift+Tab switch (other than into plan mode) is invisible to all of these, which is why the roster states the mode and where it came from.

**Effort** — `--effort`, else `$CLAUDE_EFFORT`, else `high`. Clamp to the provider and note any clamp:

- **codex** — the model's `supported_reasoning_levels`.
- **grok** — read `models_cache.json` next to the `user` layer path in `grok inspect --json` → `.configSources.layers` (it follows wrappers that set `GROK_HOME`; default `~/.grok`). When `.models[<model>].info.supports_reasoning_effort` is false the flag is silently ignored: drop it and put "effort: n/a" in the roster; otherwise clamp to `.info.reasoning_efforts[].value`.

| Mode | codex flags | grok flags | claude |
|---|---|---|---|
| `plan` | `-s read-only` | `--permission-mode default --sandbox read-only` | inherits |
| `default`, `dontAsk` | `-s read-only` | `--permission-mode default` | inherits |
| `acceptEdits` | `-s workspace-write` | `--permission-mode default --allow Edit --allow Write --sandbox workspace` | inherits |
| `auto` | `--approve-for-me -c sandbox_workspace_write.network_access=true` | `--permission-mode bypassPermissions --sandbox workspace` | inherits |
| `bypassPermissions` | `--dangerously-bypass-approvals-and-sandbox` | `--permission-mode bypassPermissions` | inherits |

Verified 2026-09-10 against codex-cli 0.153.2, grok 0.2.103, Claude Code 2.1.267:

- A headless consultant has nobody to ask. Where this session would prompt the user, the consultant is denied — never silently upgraded. Codex can't split edits from commands, so `acceptEdits` confines both to the workspace with no network.
- Codex `workspace-write` blocks network unless `network_access=true`.
- Grok's `--permission-mode` flag only applies `default` and `bypassPermissions`; `plan`, `acceptEdits`, and `auto` are accepted and ignored, so those rows are rebuilt from allow rules plus a kernel sandbox profile.
- Grok headless **ends the whole run** when a call needs approval (`stopReason: "Cancelled"`) — observed, although grok's docs say the call is reported back to the model. Report "stopped at the permission wall", offer a `--mode`, don't retry silently.
- Grok runs Claude's hooks but feeds them its own payload (`toolName`, `toolInput`). A guard hook that reads Claude's `tool_name` / `tool_input` sees nothing and allows the call, since hooks fail open — don't count on it for a grok consultant that can write.
- Claude consultants are `general-purpose` Agent-tool subagents. With no `permissionMode` of their own they inherit this session's mode in every row, along with its MCP tools, permission rules, and hooks. In the background they keep the working built-ins but lose interactive ones like AskUserQuestion; their permission prompts surface in this session.

**The mapping is the access — never narrow it.** The reflex is to lock a consultant down; `/council` does that on purpose, this skill does not.

| Tempting move | Why it's wrong here |
|---|---|
| read-only sandbox "since it's only advice" | Advice vs. changes is the brief's mandate (Step 3), not a sandbox |
| `--ignore-user-config`, banning MCP/web tools, `Plan`/`Explore` agent type | Strips tools this session has; the consultant can't reproduce, run, or look anything up |
| no network "for safety" | This session has network |
| `fork` subagent for a Claude consultant | Ignores the model override and inherits your anchoring — not a second opinion |

## Step 3 — Write the brief

The consultant has seen none of this conversation. Write `$SP/brief.md` with these parts, in order — `$SP` is a fresh directory per consultation, `<session scratchpad>/consult/<provider>-<HHMMSS>`, so a later consult never overwrites an earlier one's session id. In plan mode, where you can't write files, pass the same text inline: a quoted heredoc on codex's stdin (`- <<'BRIEF'`), `-p "$(cat <<'BRIEF' … BRIEF)"` for grok, the Agent `prompt` for claude.

1. **Role** — "You are being consulted by another coding agent (Claude Code) working with a user. Investigate independently; the hypotheses below are unverified — try to refute them. Respond in a neutral technical register." (Provider configs can set a persona.)
2. **The ask** — the user's words, verbatim, plus one line saying what "this" refers to when they point at the conversation.
3. **Context** — the problem; exact errors and output, verbatim; what was tried and what happened; the current hypothesis, marked unverified; relevant paths; `git status --short`.
4. **Project rules the CLI won't load** — for codex, which reads `AGENTS.md` but not `CLAUDE.md`: copy the rules that matter (package manager, verify commands). Claude and Grok consultants load `CLAUDE.md` themselves.
5. **Mandate** — by default: "Investigate and advise. Run whatever helps (tests, builds, repro scripts), but leave project files as you found them." When the user said fix / implement / let it try: "You may change files in the workspace to fix this." Either way: no commits, pushes, or other outward-facing actions.
6. **Access** — one line on what the mapped access allows and what happens at its edge, so the consultant doesn't spend its run discovering it. Codex: blocked commands fail back to it. Grok in `plan`, `default`, or `acceptEdits`: a command that needs approval ends the run — name the verification commands instead of running them.
7. **Response contract** — "Reply with exactly these sections: Answer (first line is the verdict) · Evidence (file:line; commands run with key output) · Key assumption · Confidence (high/medium/low) · Changes made (paths, or none) · Open questions."

## Step 4 — Run

Announce the roster in one line before running:
`Consulting codex gpt-6-astra @xhigh · access: auto (~/.claude/settings.json) → approve-for-me + network · reads go to OpenAI`

`$SP`, `$REPO`, `$MODEL`, `$EFFORT`, and `$ACCESS` below stand for concrete values — shell state doesn't survive between Bash calls, so write them into every command.

When the mapped access can write, snapshot the working tree — tracked and untracked — into a throwaway index; your real index and files stay untouched:

```bash
SP="<session scratchpad>/consult/<provider>-<HHMMSS>"; mkdir -p "$SP"; REPO=$(git rev-parse --show-toplevel)
cp "$(git -C "$REPO" rev-parse --path-format=absolute --git-path index)" "$SP/index"
GIT_INDEX_FILE="$SP/index" git -C "$REPO" add -A && GIT_INDEX_FILE="$SP/index" git -C "$REPO" write-tree > "$SP/tree.before"
```

Run with Bash `run_in_background: true` — consultations routinely outlast the foreground timeout. Wait for the completion notification; never predict the answer. Leave the working tree alone until it returns: anything you edit meanwhile shows up as the consultant's change.

```bash
# codex — $ACCESS from the table; drop -m to use the config default. Answer lands
# in -o; session id is the first JSONL event: {"type":"thread.started","thread_id":"..."}
codex exec --json -C "$REPO" -m "$MODEL" -c model_reasoning_effort="$EFFORT" $ACCESS \
  -o "$SP/codex.md" - < "$SP/brief.md" > "$SP/codex.jsonl" 2> "$SP/codex.err"
```

```bash
# grok — drop -m for its default, and --reasoning-effort when effort is n/a.
# Answer at .text, session id at .sessionId; .stopReason must be "EndTurn"
grok --prompt-file "$SP/brief.md" --cwd "$REPO" -m "$MODEL" --reasoning-effort "$EFFORT" $ACCESS \
  --output-format json > "$SP/grok.json" 2> "$SP/grok.err"
```

```text
claude — Agent tool (subagents run in the background):
  subagent_type: "general-purpose"   (all tools and MCP servers, this session's mode)
  model: opus | sonnet | haiku | fable
  prompt: the brief
Keep the agent id it returns — follow-ups go there.
No effort parameter: roster says "effort: session default"; a --effort flag is ignored — say so.
Data stays with Anthropic. When the model matches this session's, say so: fresh context, same family.
```

## Step 5 — Report, verify, follow up

1. **Failure first.** Non-zero exit, an empty answer, no `turn.completed` event in codex's JSONL, or a grok `stopReason` other than `EndTurn` → report the consultation as failed or cut short, quoting the relevant stderr lines. CLIs log noise (MCP auth errors and the like) on every run, so stderr alone isn't failure. Never write its answer for it.
2. **What it changed** — whenever you snapshotted, failed and cut-short runs included: rerun the `add -A` / `write-tree` line into `$SP/tree.after`, then `git -C "$REPO" diff --stat <before> <after>`. Gitignored paths (build output, generated code) aren't covered. A mismatch with its "Changes made" is itself a finding. Undoing exactly its edits, when the user asks: `git -C "$REPO" diff --binary <after> <before> | git -C "$REPO" apply`.
3. **Present** it attributed and intact — including where it disagrees with you:

```text
gpt-6-astra (codex) · auto access · session <id>
<its Answer, faithfully>
Evidence: <its key file:line citations> · Key assumption: <…> · Confidence: <…>
Changed: <diff stat | none> · Open questions: <…>
My read: <agree / disagree — and what you checked in the code>
```

Add grok's `total_cost_usd` to the header when it's reported. Read the code its answer rests on before repeating any claim as fact. Neither the consultant nor you is the authority — the code is. Don't adopt its fix or keep its changes on its say-so; the user decides, then the project's verify gate runs.

**Follow-ups** go to the same session, run in the background from `$REPO`, with the follow-up as a new file in `$SP` (inline in plan mode). Re-snapshot first when access can write:

| Provider | Follow-up |
|---|---|
| codex | `codex exec $ACCESS resume <thread_id> --json -c model_reasoning_effort="$EFFORT" -o "$SP/codex-2.md" - < "$SP/followup.md" > "$SP/codex-2.jsonl" 2> "$SP/codex-2.err"` — repeat the first call's `$ACCESS` *before* `resume`: without it a resumed session silently drops `--approve-for-me` to approval policy `never` and loses `-c` overrides like network. Resume has no `-C`, so run it from `$REPO` |
| grok | `grok --prompt-file "$SP/followup.md" --resume <sessionId> --cwd "$REPO" -m "$MODEL" --reasoning-effort "$EFFORT" $ACCESS --output-format json > "$SP/grok-2.json" 2> "$SP/grok-2.err"` — the sandbox profile is fixed per session, but permission flags are read per process: repeat `$ACCESS` or the follow-up silently loses `--allow` rules and `bypassPermissions` |
| claude | `SendMessage` to the agent id the Agent call returned |

## Cost and data exposure

A consultation is a real session on that provider's quota; follow-ups add turns to it. Codex and Grok send what they read to OpenAI and xAI — no sandbox mode changes that. Don't consult an external provider on code you may not share with it. A first Codex run in a directory also adds a `[projects."<path>"]` trust entry to `~/.codex/config.toml`.
