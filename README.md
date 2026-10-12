# claude-agent-handoff

> A repeatable Claude Code process for turning an internal script or agent into a clean, public-shareable bundle — cleaned source, real documentation, pinned dependencies, MIT licence, smoke test, optional GitHub push.

You wrote a useful thing. It's coupled to your internal environment — custom logger, hardcoded sheet IDs, your store names, your colleagues' names, your private dependencies. You want to hand it to someone else (a client, a peer, a contractor, a stranger on the internet) without scrubbing by hand for two days.

This is that process.

---

## What it does

Drop the slash command into your Claude Code project, run `/handoff-package <path-to-your-agent>`, and Claude produces (fanning out to sub-agents only when the source is big enough to need it):

- **Cleaned source code** — your internal imports abstracted to local stubs, your env vars renamed, your hardcoded customer/account IDs replaced with placeholders
- **Real documentation** — README, SETUP (deep platform setup steps), ARCHITECTURE, ROADMAP, HANDOFF_NOTES
- **`credentials/*.example.json` templates** — what they need to fill in, never with your real values
- **`requirements.txt` pinned to exact versions**, verified in a fresh virtualenv
- **A smoke test** that imports every module without needing credentials or network
- **Scheduling files** — launchd plist + cron example
- **MIT LICENSE**, `.gitignore`, clean directory layout
- **An air-tight scrub** for personal names, brand names, internal-ecosystem references, real account IDs, secrets — verified by an independent agent + your own greps
- **Optional**: `gh repo create --private --push` for one-command GitHub publishing

Roughly 15 minutes for a small script, up to an hour for a large agent. Local-only by default — nothing pushed externally without your explicit OK.

---

## Install

Drop the slash command into your Claude Code project's commands directory:

```sh
# In the project where you want to package an agent:
mkdir -p .claude/commands
curl -fsSL https://raw.githubusercontent.com/matthew2rs/claude-agent-handoff/main/commands/handoff-package.md \
  -o .claude/commands/handoff-package.md
```

Or clone and copy:

```sh
git clone https://github.com/matthew2rs/claude-agent-handoff.git
cp claude-agent-handoff/commands/handoff-package.md /path/to/your/project/.claude/commands/
```

To have it in every project instead, put it in `~/.claude/commands/` rather than the project's `.claude/commands/`.

Restart Claude Code (or just open a new chat in the project). The slash command appears as `/handoff-package`.

---

## Usage

In a fresh Claude Code chat inside your project:

```
/handoff-package agents/my-cool-thing
```

or

```
/handoff-package scripts/order-monitor.py
```

Claude asks two scoping questions (source confirmation, bundle name), then runs the full pipeline. You get a bundle at `handoff-bundles/<name>/`. Review it. When you're satisfied, give the go to push to GitHub (or skip GitHub entirely and zip the folder).

---

## What it expects

- **Python is the best-tested path** for the smoke test, pinning and round-trip steps. Other languages work too: the command falls back to that language's lockfile and clean install.
- **Claude Code with WebSearch** and the standard tools. It uses the Workflow tool for the parallel steps if you have it, and plain sub-agents if you don't.
- **`gh` CLI authenticated** (only for the optional GitHub push step — skippable).
- A source agent that actually does something — the process is for production-shape tools, not 50-line throwaways.

---

## Philosophy

A handoff bundle is not "your code with the credentials removed." It is a different artefact, written for a different audience, with documentation a stranger could follow on a fresh machine. The pipeline forces that shift.

Three rules hold this together:

1. **NO publishing without explicit user OK.** Local-only until you ship.
2. **Trust-but-verify.** Every agent's "I scrubbed X" gets an independent grep pass. The verifier will miss things — extra passes catch the rest.
3. **The bundle is a separate world.** Generic env vars, no internal imports, no references to your internal infrastructure. The reader has never met you and has only the same kind of API environment.

---

## Reference build

The pattern was distilled from packaging a funnel-aware paid-ads optimiser (production cron-driven Python agent across Google + Meta + Bing). End result: a 58-file, 808KB, MIT-licensed bundle with verified pinned requirements on Python 3.13. Built in one Claude Code session using exactly this slash command. The command file includes the target folder layout.

When in doubt about scope, length, or polish — match that bar.

---

## Licence

MIT. Use it, fork it, modify it, build a business on top of it. Just keep the copyright notice.
