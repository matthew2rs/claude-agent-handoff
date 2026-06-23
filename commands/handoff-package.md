---
name: handoff-package
description: Package an internal agent into a public-safe, reproducible handoff bundle — cleaned source, docs, examples, MIT licence, pinned deps, smoke test, optional GitHub push.
argument-hint: "<source-path> (e.g. 'agents/my-thing' or 'scripts/order-monitor.py')"
---

# /handoff-package — internal agent to clean public bundle

A reusable multi-agent process for turning an internal script/agent into a clean handoff bundle a stranger could clone, install, and run. Run inside Claude Code; pass the agent's directory or file as the argument.

## The one rule

**Build the bundle locally. Do not publish to GitHub or anywhere external without explicit user approval.** Air-tight scrubbing reduces leak risk but the final visibility call belongs to the human.

## How it runs

### 0. Scope it

Two questions before starting:
- **Source path** — where does the agent live? (e.g. `agents/my-thing`, `scripts/order-monitor.py`)
- **Bundle name** — the public-facing name. Avoid `claude-*` prefix unless an LLM is genuinely in the decision path — it misleads.

### 1. Prior-art check (do this FIRST, it affects everything)

WebSearch GitHub for existing projects in the same space. Distinguish SHAPE: most existing public agents are conversational Claude Code skills; if yours is a production cron-driven Python daemon, it is a different category. Report to user — they decide novel-vs-fork.

### 2. Discovery (parallel: Inventory + Security)

Two agents in a Workflow:

- **Inventory** — file manifest, dependency graph, hardcoded constants, internal-ecosystem integration points (custom logging, custom notification, custom HTTP helpers, custom data sources). Returns structured output.
- **Security** — grep for hardcoded secrets, real account IDs (Google customer_id, Meta `act_NNNN`, Bing customer_id, Stripe acct, etc.), real Sheet IDs, real domains, PII. NEVER include actual secret values in output — redacted excerpts only.

### 3. Quality + Documentation (parallel)

- **Code quality sweep** — pyflakes/ruff (if Python), type-hint coverage, dead code, TODO/FIXME classification, fresh-import check (would modules import without internal-ecosystem deps?).
- **Documentation agent** (the deepest job — biggest spend) — writes:
  - `README.md` — elevator pitch, who it is for, what it does, what it does NOT do, quick-start, licence
  - `SETUP.md` — deep platform setup. If touching paid-ads APIs: Meta Business Manager + system user + long-lived token, Google Ads MCC + developer token tier escalation, Bing OAuth, Shopify Admin API scopes + version pinning, Google Sheets service account + sharing pattern, launchd/cron wiring
  - `ARCHITECTURE.md` — design at a high level, ideally with a mermaid diagram
  - `ROADMAP.md` — queued, intentional limitations, blocked-on-X
  - `HANDOFF_NOTES.md` — honest list of what works, what is parked, where dependencies are tightest, what is untested, open decisions for the user

### 4. Gap test (fresh eyes)

One independent agent that has NOT seen prior steps. Reads only the bundle's docs. Asks: can a stranger on a clean Linux box, given only this bundle + API tokens, reproduce a working install? Returns blockers + minimal-smoke-test recommendation.

### 5. Apply fixes + Assemble bundle

Single agent: copies cleaned source into `bundles/<name>/src/`, copies any policy/canon docs into `policy/`, redacts identifiers from security report, writes `credentials/*.example.json` templates with `<YOUR_X>` placeholders, scheduling files (launchd plist + cron example), smoke test that imports every module without network/credential access, `.gitignore`, `requirements.txt`. Addresses gap-test blockers.

### 6. Air-tight scrub pass (mandatory — 2-agent workflow)

The previous pass produces a structurally-correct bundle but personal/brand names slip through. Run a SECOND workflow:

- **Scrub agent** applies a comprehensive dictionary across every file:
  - Real store/brand names → `Store A / B / C` (or chosen placeholders)
  - Internal slugs (e.g. `prod`, `staging`, `client_a`) → generic `store_a/b/c`
  - Personal first names → role descriptions (`the operator`, `the accountant`, `the content manager`)
  - Internal env vars (e.g. `MYCO_*`) → bundle-namespaced env vars
  - Internal imports (`internal.logger`, `internal.notify`, `internal.http`) → bundle-local stubs at `src/utils/`
  - Internal-product references (a central spreadsheet, an internal dashboard, an internal lesson library) → generalised abstractions
  - Region/timezone hardcodes — leave code, document as user-configurable in SETUP § Localisation
  - Real incident anecdotes ("after Bob deleted everything") → generalised ("after a destructive accident")
  - Specific real-customer mentions in code (e.g. a major vendor exclusion) → generic env-var-driven abstraction

- **Independent verifier** — grep sweeps, no knowledge of scrub. Reports residual hits by category and severity. Fix any leaks before continuing.

Trust-but-verify with your own greps too. The verifier WILL miss things — extra independent passes catch the rest.

### 7. Pin requirements + fresh-machine round-trip

Mandatory for a professional handoff:

1. Copy bundle to `/tmp/<name>-roundtrip/`
2. `python3 -m venv .venv && .venv/bin/pip install -r requirements.txt`
3. Run smoke test in the fresh venv — must pass
4. `.venv/bin/pip freeze` → capture exact versions of every dep (direct + transitive)
5. Rewrite `requirements.txt` with `==` pins on every package
6. SECOND round-trip in a fresh venv with the pinned requirements — `pip freeze` must match the file EXACTLY
7. Smoke test passes again
8. Update SETUP.md to note: "Tested on Python X.Y in a clean virtualenv"

### 8. Add LICENSE

Default: MIT. Standard text, `Copyright (c) <year> <user's name>`. Drop at bundle root.

### 9. Final independent leak scan + smoke test

Run your own greps for these 9 categories:
1. Real brand / store / product names
2. Personal first names
3. Internal ecosystem references (env vars, internal module paths, internal sheet IDs)
4. Curriculum / playbook attribution references
5. Env-var leftovers from the rename pass
6. Slug inconsistencies (mixed old + new slugs)
7. Real Google / Meta / Bing / Stripe / Shopify identifiers
8. Secrets (`AIza`, `EAAB`, `ya29.`, `sk-`, `act_`, `customers/{10digits}`, `-----BEGIN PRIVATE KEY-----`)
9. People-incident anecdotes ("after X did Y")

ZERO hits across all 9 before considering it shippable.

### 10. Optional — git init + GitHub push (only on user's go)

```sh
cd bundles/<name>
git init -b main
git add .
git commit -m "Initial commit: <one-line elevator pitch>"
gh repo create <name> --private --source=. --push --description "<one-sentence value prop>"
```

**Default visibility: private.** Public is a one-way door — even if flipped back, mirrors/archives capture immediately. Recommend private until at least one outside install has validated the bundle.

## Principles that hold this together

- **NO publishing without explicit user OK.** Local-only until they say ship.
- **Trust-but-verify.** Every workflow agent's "I scrubbed X" gets your own independent grep. The verifier will miss things — independent passes catch the rest.
- **The bundle is a separate world.** Generic env vars, no internal imports, no references to internal infrastructure. Reader has never met the operator and has only the same kind of API environment.
- **Real production tools deserve real documentation.** SETUP.md is the deepest job — name menu paths, call out gotchas, give the recipient a fighting chance to reproduce.
- **Pin everything, round-trip twice.** Unpinned requirements drift; pinned + verified-in-a-fresh-venv is the only "reproducible" that holds up.
- **Air-tight ≠ adequate.** Bundles get reused, forked, scrutinised. The bar is professional, not "good enough for the immediate recipient."

## Reference build

Worked example for the funnel-aware-ads-optimiser, packaged from a private multi-channel ads agent into a 58-file, 808KB, MIT-licensed, Python 3.13-verified bundle in one Claude Code session. See [github.com/matthew2rs/funnel-aware-ads-optimiser](https://github.com/matthew2rs/funnel-aware-ads-optimiser) (private — sample structure).

When in doubt about scope, length, or polish — match that example.
