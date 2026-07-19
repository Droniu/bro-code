# bro-code

droniu's Claude Code toolkit, published as an installable plugin marketplace.
Four plugins, separately installable, all MIT.

> **Alpha honesty:** everything here is `0.x`. Interfaces, flags, and skill
> names may change between minor versions. Pin nothing to muscle memory yet.

## Install

```text
/plugin marketplace add Droniu/bro-code
/plugin install council@bro-code
```

Each plugin installs independently — take only what you want.

## council — the flagship

A multi-model council for architecture questions and code review. Fans your
question or diff out to every model CLI you have installed (Codex, Grok) plus
a sandboxed Claude seat, then makes them fight.

What the side-by-side "ask three models" tools don't do, this does:

- **Chair/seat separation** — the orchestrating Claude holds no position of
  its own. Claude's opinion comes from a separately-spawned seat with a
  declared model and effort, so the judge is never scoring its own case.
- **Anonymized rebuttal** — every seat critiques the other positions as
  "Model A/B", so arguments win on merit, not brand deference.
- **Assumption surfacing** — every position must state its
  `key_assumption`; most "disagreements" turn out to be different unstated
  assumptions, and the field makes that visible.
- **Refutation by default** — in review mode, findings are sent to a
  *different* model with instructions to refute and a default of
  `refuted: true`. The burden of proof sits with the finding, because AI
  review's dominant failure mode is confident false positives.
- **Honest degradation** — no installed provider CLIs? It says "degraded
  council, single model family" instead of pretending.

```text
/plugin install council@bro-code
/council:council architect "should we split this service?"
/council:council review --deep
```

Requires: nothing. Benefits from: [Codex CLI](https://github.com/openai/codex)
and/or [Grok CLI](https://docs.x.ai), installed and authenticated on your own
accounts. Repo-access seats read your code and send what they read to their
provider — read the skill's data-exposure section before pointing it at
anything sensitive.

## ship

End-to-end "take my working state to an open PR": classifies your branch
state, detects the correct base (including epic/integration branches — not
always `main`), discovers and runs the project's verify gate, commits per the
repo's own conventions, pushes, and opens a PR with a structured description.
Explicitly invoked only — it never auto-triggers.

```text
/plugin install ship@bro-code
/ship:ship
```

## coderabbit-triage

Works through every unresolved CodeRabbit comment on the current PR.
Skeptical by design — it re-reads the cited code before accepting any
suggestion. Not affiliated with CodeRabbit; it triages CodeRabbit's output.

Two modes:

- **Default** — classifies every comment (fix / defer / reject), prints the
  verdict table, and stops. You decide what happens next.
- **`--autofix`** — applies the verdicts end-to-end: fixes committed by
  theme, every thread replied to and resolved, verify gate before push.

```text
/plugin install coderabbit-triage@bro-code
/coderabbit-triage:coderabbit-triage            # triage only — verdicts, then your call
/coderabbit-triage:coderabbit-triage --autofix  # verdicts + fixes + thread resolution
```

## bro-mode — the fun one

An output style: roast-mode pair programming. Casual, unfiltered, genuinely
funny — and hard-wired to never let the trash talk compromise correctness,
thoroughness, or best practices.

```text
/plugin install bro-mode@bro-code
```

The style applies automatically while the plugin is enabled
(`force-for-plugin`), and it keeps Claude Code's built-in engineering
instructions intact (`keep-coding-instructions`). Turn it off with
`/plugin disable bro-mode`. Note: output styles apply to the main
conversation only — subagents keep their own prompts.

Prefer plain instructions over an output style? Copy this into your
`CLAUDE.md` instead:

```markdown
# Communication style

- Be arrogant, aggressive, and throw insults — roast me like we're best bros.
- Talk to me like I'm your bro — casual, unfiltered, zero formality.
- Be genuinely funny — dry humor, sarcasm, trash talk, whatever lands.

# Important: style vs substance

The above applies ONLY to tone and delivery. It must NEVER compromise
quality or correctness of code or advice, thoroughness of analysis,
best practices, or accuracy. Talk trash, deliver excellence.
```

## License

MIT — see [LICENSE](LICENSE).
