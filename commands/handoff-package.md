---
name: handoff-package
description: Package an internal agent into a public-safe, reproducible handoff bundle — cleaned source, docs, examples, MIT licence, pinned deps, smoke test, optional GitHub push.
argument-hint: "<source-path> (e.g. 'agents/my-thing' or 'scripts/order-monitor.py')"
---

# /handoff-package — internal agent to clean public bundle

A reusable process for turning an internal script/agent into a clean handoff bundle a stranger could clone, install, and run. Run inside Claude Code; pass the agent's directory or file as the argument. The bundle lands at `handoff-bundles/<name>/` in the current project.

## The one rule

**Build the bundle locally. Do not publish to GitHub or anywhere external without explicit user approval.** Air-tight scrubbing reduces leak risk but the final visibility call belongs to the human.

## Sizing: who does the work

- **Small source** (under ~500 lines and ~10 files): do every step yourself. Spawning agents for a script this size costs far more than it saves.
- **Larger source**: fan out where marked below. If the **Workflow** tool is available, use it (invoking this command counts as the opt-in); otherwise launch **Agent** sub-agents in one message so they run concurrently. Sub-agents have not seen this conversation, so every prompt must be self-contained and must point at the design spec (step 3) rather than restating the design.

In both cases **you, the orchestrator, own the design** and run the final checks yourself.

## How it runs

### 0. Scope it

Two questions, skipping any the argument already answers:
- **Source path** — where does the agent live?
- **Bundle name** — the public-facing name. Avoid a `claude-*` prefix unless an LLM is genuinely in the decision path; it misleads.

If running non-interactively, use the folder name in kebab-case (`stock_alert` → `stock-alert`) and carry on.

Then, without asking, build the **scrub dictionary**: every real identifier that must not appear in the bundle. Sources:
- the user's name and email (`git config user.name` / `user.email`)
- every author in `git log --format='%an %ae' -- <source>`
- the remote's org/repo name (`git remote -v`), if there is one
- company, brand, store, product, colleague and customer names in the source, its README, its git log messages and any project-level `CLAUDE.md`
- domains, hostnames, absolute paths (`/Users/<name>/`, `/home/<name>/`), and internal env-var prefixes

Write it to `handoff-bundles/<name>.scrub.txt`, **outside** the bundle folder, never inside it. Grep short entries (first names, `/Users/`) with word boundaries so they don't drown in false positives.

### 1. Prior-art check

Two or three WebSearches for existing projects in the same space. Distinguish SHAPE: most public agents are conversational Claude Code skills; a cron-driven daemon is a different category. If a close match exists, report it and let the user decide novel-vs-fork. If nothing close exists, say so in one line and carry on.

### 2. Discovery: inventory + security (fan out on larger sources)

- **Inventory** — file manifest, dependency graph (declared vs actually imported; flag unused deps), hardcoded constants, internal integration points (custom logging, notification, HTTP helpers, data sources), and real bugs worth fixing (pagination, crash paths, import-path assumptions).
- **Security** — hardcoded secrets, real account/customer IDs, spreadsheet IDs, domains, PII, author identities in history. Never put actual secret values in output: redacted excerpts only. If a live secret is found in the source, tell the user privately at the end that it should be rotated. That note belongs in your report, not in the bundle.

### 3. Design spec (you write this, not an agent)

Write `handoff-bundles/<name>.design.md` (next to the bundle, not inside it) before any code or docs exist. It is the contract every later step builds against, so docs and code can't drift apart:
- target layout (below), package name, entry point
- bundle env-var prefix (e.g. `SSA_*`) and the full list of env vars
- config precedence (env var > credentials file > default)
- which internal imports become local stubs, and what each stub does
- **copy-and-clean vs rewrite**: copy when the code is sound; rewrite when cleaning would carry bugs over. Fixing real bugs is in scope. List every fix for HANDOFF_NOTES.
- dependencies to keep and drop

Target layout:

```
handoff-bundles/<name>/
├── README.md  SETUP.md  ARCHITECTURE.md  ROADMAP.md  HANDOFF_NOTES.md
├── LICENSE  .gitignore  requirements.txt (or the language's lockfile)
├── src/<package>/       cleaned source, with __init__.py files
│   └── utils/           local stubs replacing internal imports
├── policy/              rules/design docs the agent reads at runtime, if any
├── credentials/         *.example.json templates with <YOUR_X> placeholders only
├── scheduling/          launchd plist + cron example, if it runs on a schedule
└── tests/smoke_test.*   imports every module and exercises the core logic with
                         network blocked and no credentials
```

Never copy `.git`, `.env`, real credential files, logs, caches or virtualenvs into the bundle.

### 4. Build the code

Produce `src/`, `credentials/`, `scheduling/`, `tests/`, `.gitignore`, the dependency file and the LICENSE (step 8) per the design spec. The smoke test imports every module except entry-point shims like `__main__.py`. Run it. It must pass before docs are written.

### 5. Write the docs (fan out on larger sources)

Written against the code that now exists:
- `README.md` — elevator pitch, who it's for, what it does and does NOT do, quick-start, licence
- `SETUP.md` — the deepest job. For **every external platform the agent touches**: where to get the credential (exact menu paths), the scopes/permissions it needs, rate limits, API-version pinning, and the gotchas. Then scheduling (launchd/cron) and localisation (timezones, currencies, regions as configurable)
- `ARCHITECTURE.md` — design at a high level, ideally with a mermaid diagram
- `ROADMAP.md` — queued work, intentional limitations, blocked-on-X
- `HANDOFF_NOTES.md` — honest list: what works, what's untested (be specific, e.g. "never called against a live store"), bugs fixed vs the original, where the abstraction seams are, open decisions

### 6. Gap test (fresh eyes)

One independent sub-agent that has not seen the earlier steps. Use one even on a small source: a re-read by you isn't fresh eyes. It reads **the built bundle**, docs and code, and answers: can a stranger on a clean Linux box, given only this bundle and their own API tokens, reproduce a working install? Do the docs match the code? Fix every blocker it finds.

### 7. Scrub, then verify (sequential, never parallel)

1. **Scrub** — apply the dictionary across every file, including hidden files. If the code was rewritten rather than copied, there's less to scrub, but the docs and comments still need it:
   - real brand/store/company names → neutral placeholders (`Store A`, `example-shop`)
   - personal names → roles (`the operator`, `the accountant`)
   - internal env vars → the bundle prefix
   - internal imports → the local stubs
   - internal products (a central spreadsheet, an internal dashboard) → general descriptions
   - real incident anecdotes ("after Bob deleted everything") → general ones ("after a destructive accident")
   - hardcoded real-customer special cases → env-var-driven options
   - timezone/region hardcodes → leave in code, document as configurable
2. **Verify** — only after the scrub finishes, and wait for its result before step 10, an independent agent with no knowledge of the scrub greps for every dictionary entry plus the categories in step 10, and reports residual hits. Fix them.

### 8. LICENSE (written in step 4)

Default MIT, `Copyright (c) <year> <holder>`. The holder is the user's name unless they name a company or handle. **The LICENSE copyright line is the one place a real name is allowed**; it is exempt from the scrub. If running non-interactively, use `git config user.name` and flag it in the report so the user can change it.

### 9. Pin dependencies + fresh-machine round-trip

Run this in a fresh temporary directory outside the project (`mktemp -d`, or your session's scratch directory if you have one).

Python:
1. Copy the bundle in. `python3 -m venv .venv && .venv/bin/pip install -r requirements.txt`
2. Run the smoke test. It must pass.
3. `.venv/bin/pip freeze` → rewrite `requirements.txt` with `==` pins on every package, direct and transitive.
4. Second round-trip in another fresh venv with the pinned file. `pip freeze` must equal the file's non-comment lines. Smoke test passes again.
5. Note in SETUP.md: "Tested on Python X.Y in a clean virtualenv".

**No third-party dependencies?** Write a `requirements.txt` holding only a comment saying so, still do the fresh-venv round-trip to prove it runs on the standard library, and say so in SETUP.md.

Other languages: commit the lockfile (`package-lock.json`, `Cargo.lock`, `go.sum`…), install from it in the clean copy (`npm ci`, `cargo build --locked`, `go build`), run the smoke test there, note the toolchain version.

### 10. Final leak scan (your own greps, not an agent's)

Across the whole bundle, hidden files included:
1. every entry in the scrub dictionary (case-insensitive)
2. brand / store / product / company names
3. personal names and emails
4. internal ecosystem references: env-var prefixes, internal module paths, spreadsheet IDs, absolute home paths
5. leftover old env vars or slugs next to their renamed versions
6. real platform identifiers: ad-account and customer IDs, `*.myshopify.com` domains that aren't placeholders, 10-digit customer IDs, Stripe `acct_`
7. secrets: `-----BEGIN .*PRIVATE KEY-----`, `AKIA[0-9A-Z]{16}`, `AIza`, `ya29.`, `EAAB`, `xox[abp]-`, `ghp_`/`github_pat_`, `sk-`/`sk_live_`/`rk_live_`, `shpat_`/`shpca_`/`shpss_` followed by hex, `Bearer [A-Za-z0-9._-]{20,}`
8. people-incident anecdotes ("after X did Y")

ZERO real hits, apart from the LICENSE holder line, before calling it shippable. A pattern hit on a placeholder (`example-shop.myshopify.com`, `example.com`) or an ordinary identifier is a false positive: look at each one and say so in the report, don't ignore it silently. Then delete `handoff-bundles/<name>.scrub.txt` and `handoff-bundles/<name>.design.md`.

Report to the user: file count and size, the smoke-test result and runtime version, the LICENSE holder used, anything rotated or still to rotate, what's open in HANDOFF_NOTES, and the visibility you'd recommend. **Stop here and wait for their go.** Do not run step 11 on your own.

### 11. Optional — git init + GitHub push (only on the user's explicit go)

```sh
cd handoff-bundles/<name>
git init -b main
git add .
git commit -m "Initial commit: <one-line elevator pitch>"
gh repo create <name> --private --source=. --push --description "<one-sentence value prop>"
```

**Default visibility: private.** Public is a one-way door: even if flipped back, mirrors and archives capture it immediately. Recommend private until at least one outside install has validated the bundle.

## Principles that hold this together

- **NO publishing without explicit user OK.** Local-only until they say ship.
- **The orchestrator owns the design.** One design spec, written before code and docs, so they can't drift.
- **Trust-but-verify.** Every "I scrubbed X" gets an independent grep, and your own. The verifier will miss things; independent passes catch the rest.
- **The bundle is a separate world.** Generic env vars, no internal imports, no references to internal infrastructure. The reader has never met the operator.
- **Real production tools deserve real documentation.** SETUP.md names menu paths and gotchas.
- **Pin everything, round-trip twice.** Pinned and verified in a fresh environment is the only "reproducible" that holds up.
- **Air-tight ≠ adequate.** Bundles get reused, forked, scrutinised. The bar is professional.

## Reference build

Distilled from packaging a production multi-channel paid-ads agent (cron-driven Python, Google + Meta + Bing) into a 58-file, 808KB, MIT-licensed, Python 3.13-verified bundle in one Claude Code session. Also test-run on a 45-line Shopify stock-alert script: 19 files, smoke test passing on Python 3.9 and 3.14, zero leaks from a planted set of names, IDs and a live-format token.
