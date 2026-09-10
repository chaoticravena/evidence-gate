# evidence-gate

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Pre-production audit that scans a codebase across 8 fixed categories before anything ships. Built on one premise that most audit skills skip: a "pass" without evidence is worse than no audit, because it creates false confidence. This skill requires proof for every category marked clean — a test run, a real request against a real environment, an actual command output — not a summary of what the code looked like on read-through.

## How this differs from a generic ship-gate/quality-gate skill

Most pre-deploy audit skills report pass/fail/manual per category based on reading the code. That's useful for catching obvious misses, but it silently converts "I read this and didn't see a problem" into "this is verified" — which is a different claim. evidence-gate keeps those two states separate: a category is only "✅ verified" if something was actually run and produced a real result; otherwise it's "reviewed, não testado" and stays visibly unverified, even if nothing suspicious turned up. The audit never quietly upgrades a read-through into a pass.

See [SKILL.md](.claude/skills/evidence-gate/SKILL.md) for the full behavior spec: intercept triggers, the 8 categories, verification discipline, and output format.

## Installation

Project-level (shared via the repo, recommended for teams):

```
git clone https://github.com/chaoticravena/evidence-gate.git <your-repo>/.claude/skills/evidence-gate
```

Commit `.claude/skills/evidence-gate/` into your repo so collaborators get it automatically.

User-level (available across all your projects):

```
git clone https://github.com/chaoticravena/evidence-gate.git ~/.claude/skills/evidence-gate
```

After installing, restart your Claude Code session so it's discovered. Invoke with `/evidence-gate`, or let it trigger automatically on deploy-intent phrases.

## Usage

- `/evidence-gate` — run the full audit now.
- `/evidence-gate [category]` — run a single category (quick recheck only, not a substitute for a full run).
- `/evidence-gate diff` — run all 8 categories, scoped to files changed since the last commit/deploy marker.

## See also

[blind-spot-hunt](https://github.com/chaoticravena/blind-spot-hunt) — a complementary skill for after evidence-gate: a hypothesis-driven adversarial pass for business-logic bugs outside evidence-gate's 8 fixed categories.

## License

MIT — free to use, fork, and modify. If you build on this, a link back is appreciated but not required.

## Changelog

- 1.0.0 — initial release. 8 fixed categories, evidence-required verification discipline, stack detection, time-boxing, full/diff/single-category run modes.
