
# genchi 🕵️

![genchi re-fetches real state: 0 rows against an expected 45, then verified at 45](docs/hero.svg)

> Part of a set of zero-dependency CI tools for AI-agent repos — start with **[reflint](https://github.com/hyuga611/reflint)**.

**Nothing gets to report "done" except a re-fetched real result.** A completion verification gate for AI agents and automation — framework-agnostic, zero-dependency, and it runs no LLM.

**「完了しました」を、再取得した実結果でしか名乗らせない。** AIエージェント/自動化のための完了検証ゲート。

[![npm](https://img.shields.io/npm/v/@hyuga/genchi.svg)](https://www.npmjs.com/package/@hyuga/genchi)
[![license](https://img.shields.io/badge/license-MIT-blue)](LICENSE)
[![zero dependencies](https://img.shields.io/badge/dependencies-0-brightgreen)](package.json)

```bash
npx @hyuga/genchi verify --probe "psql -tAc 'select count(*) from t where batch=123'" --count 45
```

## Why

When you hand real work to an agent, the scariest hallucination isn't a wrong sentence. It's **the fabrication of having done the work at all.**

> "Inserted 45 rows. Done." — then you open the admin panel and not a single row landed.

The cause is that *acting* and *checking* are the same step. The agent reads a tool's return value, generates the next sentence, and **claims completion without ever looking at the world it just changed.** Having never looked, it can't notice the failure either.

`genchi` (現地現物 — *go and see the actual thing*) separates the two. **An operation with side effects is not reported as complete until a separate probe has run and what it returned has been checked against an expectation.** Empty, error and mismatch are refusals, and the evidence shown is what the probe returned — never something the model said about it.

What that does *not* do is guarantee the probe looked at anything. See [What this does not buy](#what-this-does-not-buy).

## This is not a linter — it works at runtime

[reflint](https://github.com/hyuga611/reflint) (do references resolve), [skills-lint](https://github.com/hyuga611/skills-lint) (do skills collide), and [carrylint](https://github.com/hyuga611/carrylint) (is it portable) all inspect config files **statically**. genchi doesn't. It runs **at runtime, after the action, and re-reads the state of the world to compare against the claim.**

- guardrails / deepeval / promptfoo — verify the **text** an LLM produced
- **genchi** — re-fetches the **state of the world** after the action and checks the report against it

"Inserted 45 rows" is flawless as text. What's wrong isn't the text — it's the world behind it. That's why no amount of grading the text catches it.

## Use as a library

```js
import { gate, verify, expect } from '@hyuga/genchi';

// the side effect: insert 45 rows
await db.insert(rows);

// before claiming done, re-fetch real state with a separate call.
// the point is that probe RE-READS the world — it is not the action's return value.
await gate({
  action: 'insert 45 rows',
  probe: () => db.count({ where: { batch: 123 } }), // ← re-fetches real state
  expect: expect.count(45),
});
// reaching this line means the probe was called and returned 45. Otherwise it threw.
```

`gate()` throws `GenchiIncomplete` unless the probe's result passes, so **"done" is unreachable without something having been re-read and checked**. If you want the verdict without the throw, use `verify()`:

```js
const v = await verify({ action: 'upload', probe: () => fetchStatus(url), expect: expect.contains('200') });
if (!v.ok) {
  // don't paper over empty or failed results — report them as they are
  console.error(`incomplete (${v.reason}) — re-fetched: ${v.evidence}`);
}
```

### The design constraint that matters

`verify` / `gate` **accept only a probe** — the evidence has to come from calling something, not from a value you hand in alongside the claim. Omit the probe and it throws `TypeError`. That puts the re-read in its own expression, written on purpose, at the moment the completion is asserted.

Empty results and errors are never swallowed. If `probe` throws, it is reported **as-is** with `reason: 'probe-error'` — never imagined into a success. A returned `count` of 0 (nothing landed) is incomplete too. Note that genchi applies **no timeout of its own**: a probe that never settles hangs the gate rather than failing it, so put the timeout in the probe (`curl --max-time`, a statement timeout) where you need one.

### What this does not buy

**Whether the probe read anything.** A probe is a function, and there is no way in JavaScript to make a function do I/O. This passes:

```js
const result = await doTheInsert();                 // suppose nothing landed
await verify({ action: 'insert 45 rows',
               probe: () => result.inserted,        // the action's own return value
               expect: expect.count(45) });         // → ok: true
```

Until 0.3.0 this README said that was "structurally impossible" and "unwritable". It is one line, and the CLI printed `re-fetched: 45` about a `--probe "echo 45"` that re-fetched nothing — this library asserting, in its own output, a thing it had not checked. That is the failure it exists to prevent, so the wording is now what it can actually stand behind: *the probe returned*.

**Whether the read was independent of the write.** A probe that genuinely re-fetches can still
re-fetch through the same client, transaction, or cache that produced the misleading result in the
first place. A transaction reading its own uncommitted writes, a cache in front of the store, a
stale read replica, a bug in the client itself — in each case the read is real and still agrees with
a write that did not land. Reading through an independent path *reduces* that: a different client,
plain `curl` instead of the SDK, the database CLI instead of the ORM. It does not eliminate it —
they may still share a backend, a replica, or the same credentials. genchi cannot enforce any of
this: a probe is a function, and the library cannot see which connection it used. Treat path
independence as a practice, and spend it where the write path is the part you doubt.

What remains true is worth having, and it is not nothing:

- the re-read is a separate expression you have to write, rather than a field you fill in
- empty, error and mismatch are refusals, not silently-passed successes
- the evidence is the probe's own output, never a summary of it

The rest is yours to keep: **point the probe at the thing that reads the real state.** `--probe "echo 45"` will pass, and it will have been your hand that wrote it.

### Built-in expectations

| | Passes when |
|---|---|
| `expect.nonEmpty()` | real state is non-empty (default) |
| `expect.count(n)` | the count equals `n` (e.g. rows inserted) |
| `expect.atLeast(n)` | the count is `n` or more |
| `expect.contains(s)` | the state contains string `s` |
| `expect.equals(v)` | it equals `v` (strings compared trimmed) |
| `expect.matches(re)` | it matches the regex |

Every verdict names the question it asked, as `expectation` — `count(45)`, `contains("200")`,
`custom` for your own predicate, or `nonEmpty (default)` when you passed no expectation at all.
That last one is the weakest question there is: **anything non-empty passes it.** A pass under it
is not the same evidence as a pass under `count(45)`, so it no longer looks the same in the CLI,
in `--json`, or in the `GenchiIncomplete` message.

```
$ genchi verify --probe "curl -s $URL"
✓ verified [nonEmpty (default)] — the probe returned: "…"
  Note: no expectation was given, so any non-empty output passes. Pass --count/--contains/--matches to ask a real question.
```

What that does not do is tell you whether the expectation was the *right* question. A screenshot
that arrived downscaled past legibility, a truncated log tail, a page fetched before it finished
rendering — all of them are non-empty and well-formed, and no general gate knows what the evidence
was supposed to show. Encoding that stays yours; the gate's job is to stop the weak question from
passing as a strong one in the output.

### Make the evidence assertable

An expectation is only as strong as what the probe hands back. Return the artifact and the only
question available is "did something come back". Return a **measurement of** the artifact and you
can ask a real one:

```js
// weak: the file is there, so this passes — including at 40×30 pixels
await gate({ action: 'capture', probe: () => existsSync(shot), expect: expect.nonEmpty() });

// stronger: the probe returns a number, so the expectation can be about legibility
await gate({ action: 'capture', probe: () => imageSize(shot).width, expect: expect.atLeast(1200) });
```

The same move works for a log tail (probe the line count, not the text) and for a page (probe for
the element that only exists after render, not the HTML length). It does not generalise into the
gate — "was this readable" is domain knowledge the caller has and the library doesn't — but where
the evidence has a number attached, that number is what the probe should return.

You can write your own: return `true` / `{ok:true}` to pass, `{ok:false, detail}` to fail with a reason.

## Use from the shell

Agents and scripts that don't write JS can still hand a re-fetch command to genchi. **The probe's own output is what gets emitted as evidence — nothing is invented.** (Shell output is trimmed, and evidence is JSON-encoded and truncated at 200 characters for display; what it is never replaced with is a summary of it.)

```bash
# inserted rows → count them again and check it equals 45
genchi verify --probe "psql -tAc 'select count(*) from t where batch=123'" --count 45

# uploaded a file → check the URL actually answers 200
genchi verify --probe "curl -sI https://example.com/out.png" --contains "200"

# exit 0=verified / 1=empty or mismatched / 3=probe failed (command exited non-zero)
```

Expectations: `--nonempty` (default) / `--count N` / `--at-least N` / `--contains STR` / `--equals STR` / `--matches REGEX` / `--json`.

## Wire it into Claude Code (Stop hook)

`genchi guard` re-fetches every completion contract an agent declared (one JSON object per line) and **blocks the stop with exit 2** if even one is unmet — so an agent can't end its turn on an unverified "done".

```jsonl
{"action":"insert 45 rows","probe":"psql -tAc 'select count(*) from t where batch=123'","expect":{"type":"count","value":45}}
{"action":"publish the image","probe":"curl -sI https://example.com/out.png","expect":{"type":"contains","value":"200"}}
```

```jsonc
// .claude/settings.json (excerpt)
{
  "hooks": {
    "Stop": [{ "hooks": [{ "type": "command", "command": "node ./node_modules/@hyuga/genchi/adapters/claude-code/genchi-stop-hook.mjs" }] }]
  }
}
```

See [`adapters/claude-code/`](adapters/claude-code/) for details.

## The completion contract (works as prompt text alone)

Before installing anything, dropping this paragraph into your agent's rules (`CLAUDE.md` / `AGENTS.md` / system prompt) visibly reduces false completion reports:

```markdown
## Completion contract
An operation with side effects (create, update, delete, upload, insert) may not be
reported as complete until a separate command has re-fetched the resulting state and
the raw result has been shown. Empty output, errors, and timeouts are reported as
"empty" or "failed" as-is — never filled in with an imagined id, path, or count.
```

genchi is the version of that contract enforced **by machinery instead of good intentions**.

## Design principles

- Zero dependencies, framework-agnostic, and no LLM or API key at runtime
- Never fabricate evidence — `evidence` always derives from what the probe returned (encoded, and truncated at 200 characters for display), never from a description of it
- Require a probe, so the re-read is an expression somebody wrote on purpose rather than a field filled in beside the claim — and say plainly that this is a separation to keep, not one that can be enforced

## Related tools

Zero-dependency CI linters for repos where AI agents do the work. Each one fails the PR on something that breaks quietly.

| | Catches |
| --- | --- |
| [reflint](https://github.com/hyuga611/reflint) | `AGENTS.md` / `llms.txt` / `CLAUDE.md` pointing at commands, scripts, or paths that no longer exist |
| [skills-lint](https://github.com/hyuga611/skills-lint) | `SKILL.md` broken references + `name`/trigger collisions between skills |
| [carrylint](https://github.com/hyuga611/carrylint) | Skills with the author's machine or model baked in — absolute paths, undeclared CLIs, unresolved placeholders |
| **genchi** ← you are here | Agents reporting "done" without re-fetching real-world state |
| [tracklint](https://github.com/hyuga611/tracklint) | Forms and CTAs that quietly stopped being wired for conversion tracking |
| [tokenlint](https://github.com/hyuga611/tokenlint) | Hardcoded colors that bypass your design tokens |
| [reflint for VS Code](https://github.com/hyuga611/reflint-vscode) | The same reflint checks, inline in the editor as you save |
| [orogami](https://github.com/hyuga611/orogami) | Not a linter — natural Japanese/CJK line breaking for OGP images (BudouX + font subsetting) |

## License

MIT © hyuga611
