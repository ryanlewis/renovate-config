# renovate-config

My shared Renovate config. One file: [`default.json`](default.json).

## Use it

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>ryanlewis/renovate-config"]
}
```

That's the whole per-repo config for most things. Add repo-specific grouping after it if a
repo needs it.

Two exceptions:

- Repos other people install from (`things-cli` as a Go module, `ccsesh` as a crate) should
  add `":preserveSemverRanges"`, so I'm not force-pinning versions for downstreams.
- Repos where CI doesn't really check anything should add `{"automerge": false}` for
  everything — the automerge rules below assume a green build means something.

## What it does

**Waits before taking an update.** 5 days normally, 7 for npm and PyPI. That's roughly how
long the August 2026 keyv/cacheable attack ran before anyone noticed, and npm/PyPI get the
longer wait because anyone can publish to them.

**Lets security fixes skip the queue — which needs one line most configs miss.** Renovate's
`vulnerabilityAlerts` defaults already bypass the cooldown and open a PR immediately, so
writing that block out changes nothing. The bit that's usually missing is
`osvVulnerabilityAlerts: true`, which is what tells Renovate an update *is* a security fix.

Without it you get the worst of both: real CVE fixes sit out the full 5–7 days while
nothing ever gets fast-tracked. So turn the feed on before adding cooldowns, not after.

It's also the right feed for this kind of attack — it picks up the OpenSSF malicious
package advisories as `MAL-*`, and Renovate won't offer a version flagged that way at all.

**Never runs lockfile maintenance.** Renovate ignores the cooldown during it and lets the
package manager re-resolve the whole dependency tree against live registries. That's the
one that makes a cooldown look like it's working when it isn't.

**Only automerges things that actually waited.** Renovate applies the cooldown to `major`,
`minor` and `patch` — and *not* to `pin`, `pinDigest`, `replacement`, `digest`,
`lockFileMaintenance`, `lockfileUpdate`, `rollback` or `bump`. The last rule in the file
refuses to automerge all eight of those, and it's last on purpose because later rules win.

Which leaves: runtime deps never automerge, dev deps automerge on patch after the wait, and
GitHub Actions automerge on minor/patch but **not** on digest — a digest update gets no wait
at all, and a repointed digest is how a hijacked action would reach me.

Downside: with actions pinned to SHAs, most Actions updates show up as `digest` and need
merging by hand. Worth it.

One caveat to keep honest about: an automerged Actions minor/patch repoints the pinned SHA
too — it just does it via a new tag that sat through the cooldown. For a hijacker publishing
a new minor tag, the defence is the wait, not my eyes. Same trust model as npm dev patches.

**Grouping is doing as much work as the automerge rules.** Renovate automerges a grouped PR
only if *every* update in it is automergeable, so anything that can't automerge poisons the
whole PR it lands in. That's why the groups are split the way they are:

| PR | Contains | Automerges |
| --- | --- | --- |
| all non-major dependencies | everything not claimed below | no — runtime deps sit here |
| dev dependencies (patch) | dev deps, patch only | yes, after the wait |
| github actions | Actions minor/patch | yes, after the wait |
| github action digests | Actions digest/pinDigest | no — no cooldown applies |
| one per major | a single major | no |

Left in one big batch, a runtime minor or an unwaited digest would have silently blocked
the dev patches and Action bumps that *did* wait. Splitting them is what makes the automerge
rules above actually fire.

Majors get a PR each so CI tells me whether the upgrade is survivable before I look at it.

## Two things that look pointless and aren't

`internalChecksFilter: "strict"` is already the default in Renovate 44. It's written out so
a future default change can't quietly undo it.

The `vulnerabilityAlerts` block mostly restates defaults. It's spelled out so I can read
what the policy is without going and reading Renovate's source.

## `ignoreScripts` here doesn't cover CI

`ignoreScripts: true` stops **Renovate's own** lockfile generation from running install
scripts. It does nothing for GitHub Actions — a workflow running `npm ci` still executes
them, because npm is the only one of the three that does this by default (bun and pnpm 10+
don't). Repos that install with npm in CI need their own `.npmrc` with
`ignore-scripts=true` committed.

Don't put `min-release-age` in a repo's `.npmrc` though. At machine level it's a cooldown;
in a repo it changes what everyone cloning it resolves.

## Turn on Dependabot alerts too

`osvVulnerabilityAlerts` uses the OSV database, but GitHub's Dependabot alerts are the other
half of the feed and default to off. Switch them on per repo under *Settings → Code security
→ Dependabot alerts*. Leave Dependabot *security updates* off — Renovate does the PRs.

## Checking changes

```
npx --yes --package renovate renovate-config-validator default.json
```

The validator lives inside the `renovate` package; there's no standalone one on npm. It
says `Validating default.json as global config` because `default.json` isn't a filename it
recognises as a repo config — the schema check still happens.
