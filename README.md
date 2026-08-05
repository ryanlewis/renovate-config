# renovate-config

Shared Renovate policy for the `ryanlewis` estate. One file: [`default.json`](default.json).

## Use it

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>ryanlewis/renovate-config"]
}
```

That is the whole per-repo config for most repos. Add repo-specific grouping after the
preset if needed.

Two overrides worth knowing:

- Repos that **publish for consumers** (`things-cli` as a Go module, `ccsesh` as a crate)
  should append `":preserveSemverRanges"` so downstreams are not force-pinned.
- Repos with **no CI worth trusting** should append a blanket `{"automerge": false}` rule,
  since the automerge rules below assume a green pipeline means something.

## What it does, and why

**A publish cooldown.** `minimumReleaseAge: 5 days` baseline, `7 days` for npm and PyPI.
Not an arbitrary number — it is sized against how long the 2026-08-04 keyv / cacheable
campaign ran before public detection. npm and PyPI get the longer window because they have
self-serve publish and the worst recent abuse record.

**A security fast-path that is actually reachable.** The counter-intuitive part: Renovate's
default `vulnerabilityAlerts` object *already* sets `minimumReleaseAge: null`,
`schedule: []` and `prCreation: immediate`. The promotion lane works out of the box. What
is usually missing is the **feed** — nothing tells Renovate an update is a security update.
So `osvVulnerabilityAlerts: true` is the load-bearing line here, not the
`vulnerabilityAlerts` block.

Add cooldowns without the feed and you get the opposite of the intent: real CVE fixes wait
the full window while nothing is promoted. Enable the feed *first*.

`osvVulnerabilityAlerts` is also the better feed for this attack class specifically — it
ingests the OpenSSF Malicious Packages advisories as `MAL-*`, and Renovate refuses to
propose a version flagged that way at all, rather than flagging it after the fact.

**`lockFileMaintenance` stays off.** Renovate does not enforce `minimumReleaseAge` during
lockfile maintenance — it hands resolution to the package manager, which re-resolves the
whole transitive tree against live registries. That is the specific hole that makes a
cooldown look present and not be. The same applies to `bump`, `lockfileUpdate`, `rollback`,
`pin` and `replacement`, which is why the last `packageRules` entry refuses to automerge
those update types.

**Nothing automerges without a cooldown behind it.** This is the governing rule, and it is
narrower than it first looks, because `minimumReleaseAge` does not apply to every update
type. Per Renovate's docs it is supported for `major`, `minor` and `patch`; it is **not**
supported for `pin`, `pinDigest`, `replacement` or (generally) `digest`, and
`lockFileMaintenance`, `lockfileUpdate`, `rollback` and `bump` ignore it entirely.

So the last `packageRules` entry refuses to automerge all eight of those types, and it is
last deliberately — later rules win in Renovate, so it overrides any automerge granted
above it.

What that leaves: runtime dependencies never automerge. Dev dependencies automerge on patch
only, after the cooldown. GitHub Actions automerge on `minor`/`patch` only — **not** on
`digest`, even though `helpers:pinGitHubActionDigests` pins them to SHAs, because a digest
update gets no soak time and a repointed digest is exactly how a compromised action reaches
you. Majors are parked on the dependency dashboard for a human.

The cost is honest: with actions SHA-pinned, most Actions updates arrive as `digest` and now
need a manual merge. That is the intended trade.

## Two things that look redundant and are not

`internalChecksFilter: "strict"` is already the Renovate 44 default. Setting it explicitly
is durability against a future default change, not a fix.

The `vulnerabilityAlerts` block largely restates shipped defaults. It is written out so the
promotion lane is legible in the file rather than inherited — the whole point is that you
should be able to read the policy without reading Renovate's source.

## `ignoreScripts` covers Renovate, not your CI

`ignoreScripts: true` stops **Renovate's own** lockfile generation from running dependency
install scripts. It does nothing for your GitHub Actions runners — a workflow doing
`npm ci` still executes lifecycle scripts, because npm is the one package manager that runs
them by default (bun and pnpm 10+ do not). If a repo installs with npm in CI, commit an
`.npmrc` containing `ignore-scripts=true` to that repo as well.

Do **not** put `min-release-age` in a repo `.npmrc`. At machine level it is a cooldown; in a
repo it changes dependency resolution for everyone who clones.

## Prerequisite: the feed needs switching on per repo

`osvVulnerabilityAlerts` uses the OSV database, but GitHub's Dependabot **alerts** are the
other half and are off by default. Turn them on per repo under
*Settings → Code security → Dependabot alerts*. Leave Dependabot *security updates* off —
Renovate raises the PRs.

## Validating changes

```
npx --yes --package renovate renovate-config-validator default.json
```

The validator ships inside the `renovate` package — there is no standalone
`renovate-config-validator` on npm. Note it reports `Validating default.json as global
config`, because `default.json` is not one of the filenames it recognises as a repo config;
the schema check still applies.
