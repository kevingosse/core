# dotnet-rc-watch

T-055. Watches for .NET 11 RC1 while Kevin's machine is off, and pushes to his phone
via [ntfy.sh](https://ntfy.sh). Runs entirely on GitHub's servers.

## Tiers

| Tier | ntfy priority | Trigger | Means |
|---|---|---|---|
| 1 | **urgent (5)** | a published `v11.0.0-rc*` release on `dotnet/core` | **GOSIGNAL — submit** |
| 2 | high (4) | a `release/11.0[-rc*]` **branch** appears in `dotnet/aspnetcore` | early warning — do **not** submit |
| 3 | min (1) | daily, 09:23 Europe/Paris | "still watching", proves the watcher is alive |
| 0 | high (4) | the workflow run itself failed | the watcher is **not** watching |

Per T-042: tags and branches **predict**, only a published `dotnet/core` release
**authorises**. Tier 2 deliberately never reads as "RC IS LIVE".

## Why the heartbeat exists

Without it a broken workflow is indistinguishable from a quiet week. That is the exact
failure that cost 12 hours on preview 7.

## Matchers

```
release tag   ^v11\.0\.0-rc
branch ref    ^refs/heads/release/11\.0(\.1xx)?(-rc.*)?$
```

Both are **version-anchored on purpose**. T-042 records a real false positive from an
unanchored `rc[.-]?1` grep: it matched .NET 10's `release/10.0-rc1` and fired permanently
off the wrong major version. The branch regex also has to survive
`backport/pr-63379-to-release/11.0-rc1`, which *contains* a matching substring but is
not a release branch — hence the `^`/`$` anchors.

`Self-test the matchers` runs T-042's vector table on **every** run, in both directions,
and aborts the run if any vector misbehaves. It asserts against the shipped constants
(`PROD_TAG_RE` / `PROD_BRANCH_RE`), never against a `workflow_dispatch` override, so a
drill cannot mask a broken production matcher.

## Things that are checks rather than assumptions

- **ntfy read-back.** A 200 from `curl` only proves ntfy accepted the POST. Every alert is
  re-read off the topic by message id to prove it is actually queued for the subscriber.
- **Branch-query control.** The same query is run for `release/10` and must return at least
  one RC branch. If it returns zero, the query is broken and the v11 zeros would be a dead
  grep rather than a real absence — so the run fails loudly instead of reporting "no RC".
- **Empty release list is a fault**, not "no RC".
- **Missing secret is a hard failure**, not a silent no-op.
- `prerelease` is deliberately **not** filtered: an RC carries `prerelease: true` exactly
  like a preview, so filtering it out would make this permanently silent.

## Repeat behaviour

- Tier 1 re-sends urgent every 15 min for 3 h after first detection, then goes quiet and the
  heartbeat carries it. One missed push must not lose the week; 400 pushes must not either.
- Tier 2 fires **once** per ref. Branches persist, so sightings are recorded in
  `state/branch-seen.txt` and never re-alerted.

## Setup

- Repo secret `NTFY_TOPIC` — the topic string, the only secret. Anyone who knows it can read
  the alerts and send fake ones, so it is long and random.
- Kevin's iPhone: ntfy installed, subscribed to that topic, notifications allowed and set to
  break through Focus/silent mode.

## Drills

```bash
# tier 1 positive - must push urgent
gh workflow run rc-watch.yml -f 'pattern=^v11\.0\.0-preview\.7$'

# tier 1 negative - must push nothing
gh workflow run rc-watch.yml -f 'pattern=^v11\.0\.0-rc'

# tier 2 positive - must push high
gh workflow run rc-watch.yml -f 'branch_pattern=^refs/heads/release/11\.0-preview7$'

# tier 3
gh workflow run rc-watch.yml -f heartbeat=true
```

Delete anything left in `state/` after a drill.
