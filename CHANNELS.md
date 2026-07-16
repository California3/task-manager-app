# Release & Receive Channels

This document is the authoritative spec for how Task Manager binaries,
runtimes, and plugins are published and consumed. It is intended for:

- the central build host operator (publishing side), and
- fleet operators and client code (receiving side).

## Source-of-truth roles

| Source                  | Role                                                                                          |
|-------------------------|-----------------------------------------------------------------------------------------------|
| Central build host      | **Primary source** (`TM_UPDATE_SOURCE`). Serves `manifest.json` + asset bytes over HTTP.      |
| `California3/task-manager-app` (this repo) | **Fallback source**. GitHub Releases keyed by tag prefix. Only consulted when the central source is unreachable or returns an invalid manifest. |
| `California3/task-manager` (private)       | Source code only. During the migration window it also doubles as a legacy fallback for old clients (see *Chicken-and-egg* below). After the cutoff it is no longer used for distribution. |

Clients always try the central source first. On any failure to fetch the
manifest or download the asset, they fall back to GitHub. A successful
fallback is flagged in `status()` (`from_github: true`) for observability
but is otherwise transparent to the caller.

## Publish flow (central host only)

Each of the three publish entry points (`build/publish.sh`,
`build/publish-runtime.sh`, `plugins_dist.py:publish_artifact`) performs:

1. Build the artifact and write it to the local `releases/` tree (existing
   behavior, unchanged).
2. Compute the sha256 and write a sibling `<asset>.sha256` sidecar (bare
   hex + optional newline).
3. By default (or `TM_GITHUB_PUBLISH=1`), call the shared
   `build/_github_release.sh <tag> <asset> <sidecar> <notes-file>` helper,
   which runs `gh release create` (or `gh release upload --clobber` if the
   release already exists). `--notes-file` is used (not `--notes`) so
   multi-line notes survive. Set `TM_GITHUB_PUBLISH=0` to skip GitHub upload
   (dev/test scenarios).
4. If gh is missing or auth fails, emit
   `WARN: GitHub publish skipped (...)`
   and continue. Local publish success does not depend on GitHub upload
   success.

Tag and asset naming per channel:

| Channel   | Tag                         | Asset(s)                                                       |
|-----------|-----------------------------|----------------------------------------------------------------|
| Binary    | `tm-<ver>`                  | `task-manager-<ver>`, `task-manager-<ver>.sha256`              |
| Runtime   | `runtime-<plat>-<name>-<ver>` | `<name>-<ver>.tar.gz`, `<name>-<ver>.tar.gz.sha256`          |
| Plugin    | `plugin-<plat>-<id>-<ver>`  | `<id>-<ver>.tar.gz`, `<id>-<ver>.tar.gz.sha256`                |

`<ver>` is emitted as-is by the publish scripts. Convention differs per
channel: the **TM binary** version carries a leading `v` (e.g. `v3.4.4`,
read verbatim from `dist/task-manager.version`); **Runtime / Plugin**
versions are bare (e.g. `2.1.141`, `0.1.8`, no leading `v`). Clients
strip an optional leading `v` before parsing semver, so both forms
compare correctly.

The helper asserts `gh` is installed and either `GH_TOKEN` env or `gh auth
login` stored auth is available before doing anything, so misconfiguration
fails fast with an actionable message.

## Receive flow (clients)

All three channels share one helper (`github_latest_by_prefix` /
`github_download_asset` in `updates.py`):

1. **List releases** for `California3/task-manager-app`, following
   `Link: rel="next"` pagination. per_page=100 across **all** channels, so
   capping at one page is not acceptable.
2. **Filter by tag prefix** (`tm-`, `runtime-<plat>-<name>-`,
   `plugin-<plat>-<id>-`). The caller already knows `<plat>` and `<id>`/
   `<name>`, so the prefix is built from known values — the tag is never
   parsed back into components. Exclude `prerelease==true` and `draft==true`.
3. **Parse version**: strip the channel prefix and leading `v`, parse the
   remainder with `packaging.version.Version` (lenient). Tags that fail to
   parse are logged and excluded from latest contention.
4. **Pick max semver.** This is intentional, not `published_at`, so that
   deleting a newer release rolls clients back to the next-highest.
5. **Fetch the sha256 sidecar** (`<asset>.sha256`) from the same release.
   Sidecar is mandatory: a missing sidecar aborts the fallback for that
   release.
6. **Download the asset**, verify `sha256(asset) == sidecar`. Mismatch
   raises and the client does not stage.
7. **Stage** using the existing local install path
   (`_download_and_install` for runtime/plugin; staged.json + apply for the
   binary). The GitHub-synthesized manifest is shaped to match the central
   source's manifest (`{version, sha256, exec, filename, _github_asset_url}`)
   so the install code path is reused with zero diff.
8. **ETag caching**: the release-list response's `ETag` is cached at
   `TM_DATA_DIR/gh_etag.json` and sent as `If-None-Match` on the next poll.
   GitHub returns 304 (not counted against the 60/h anonymous quota) when
   nothing changed.

### What fallback is *not*

- It is **not** a substitute for the central source. Central is faster,
  serves `manifest.json` directly, and does not impose GitHub rate limits.
- It is **not** supply-chain trusted. See README on sha256 sidecars.
- It does **not** change `obtainable()` semantics for runtimes/plugins:
  `obtainable` still reports only the central source's availability.
  Fallback is best-effort inside `ensure_*` / `_fetch_remote_manifest`.

### Prefer-fallback mode (`TM_GITHUB_PREFER`)

A client-side opt-in (env var, or Settings → Updates checkbox, default
off). When on, the client **skips the central source entirely** and
pulls all three channels (binary / runtime / plugin) directly from
this GitHub repo. Use cases: testing the fallback path without taking
central down, central known-bad / stale, debugging the GitHub consume
chain.

- `TM_GITHUB_PREFER=1` but `TM_GITHUB_REPO=""` → short-circuits back to
  central-first (misconfiguration does not leave the client empty).
- In prefer mode `obtainable()` reports the GitHub repo's availability
  rather than central's — an intentional contract extension.
- `status()` surfaces `prefer_github: true` when active. A successful
  pull still sets `from_github: true`.
- Off (default) = current behavior: central first, GitHub only on
  failure.

### Unified source race + 5-min cache (`TM_GITHUB_MIRROR`)

**Every fetch** that originally went through `TM_UPDATE_SOURCE` — lists,
manifests, version checks, and downloads, for all three channels (binary /
runtime / plugin) — now routes through a single source race. The candidate
list is `[CENTRAL, GitHub-direct, ghfast.top, gh-proxy.com, ...user_mirrors]`.
The fastest source is probed once (512KB ranged GET on the first download)
and cached for 5 minutes. Within that window every request type reuses the
same winner.

- **Mirror URL form**: `f"{prefix}/{asset_url}"` where `asset_url` is the
  `github.com/.../releases/download/...` URL (pre-302). The mirror follows
  the redirect internally and streams back. `DIRECT` (`""`) leaves the URL
  unchanged.
- **Routing**: mirror only proxies large asset downloads — it cannot proxy
  `api.github.com` (used for listing releases / fetching sidecars). When the
  cached winner is GitHub-related, list/manifest requests go to the GitHub
  API directly (HTTPS, small JSON); download requests go through the mirror.
  When the winner is CENTRAL, everything goes through central HTTP. The user
  sees "one source for 5 minutes"; the routing is internal.
- **Cold start**: when the cache is empty and a list/manifest request
  arrives first (no download URL to probe), central-first + GitHub fallback
  is used (existing behavior). The cache gets populated on the first
  download, after which all requests follow the cached winner.
- **Selection strategy**: before each download, fire a 512KB ranged GET at
  every candidate in parallel (8s wall-clock cutoff). Pick highest
  throughput; on failure or sha256-mismatch, fall through to the next
  candidate in throughput order. All candidates failing → force re-probe.
- **Security**: sha256 trust root priority = GitHub sidecar (HTTPS, fetched
  DIRECT) > central manifest sha256 (HTTP). When GitHub has the version,
  the sidecar's HTTPS-protected hash is the trust root — a malicious mirror
  cannot influence it. When GitHub lacks the version, central's HTTP hash
  is used (same as pre-change behavior). See README for details.
- **Central version pin**: download URLs carry `?ver=<ver>` so the central
  server can 404 on version skew (central ahead/behind GitHub), preventing
  bandwidth waste on mismatched downloads.
- **Observability**: `status()` surfaces `download_source_used` (last
  winner: `"CENTRAL"` / `"DIRECT"` / mirror URL) and `download_probe`
  (per-candidate KB/s from the last probe). Legacy `github_mirror_used` /
  `github_mirror_probe` fields are preserved but only reflect GitHub-label
  winners (None when CENTRAL won).
- **Probe overhead**: 3–5 parallel 512KB GETs ≈ 1.5–2.5 MB per probe,
  <2% of a 138 MB binary. Probes fire at most once per 5 min (cached).
  Downloads are infrequent, so no caching is applied beyond the 5-min
  source cache.

Off (default `TM_GITHUB_MIRROR=""`) = built-in three candidates only.
Mirrors see the public release URL being downloaded (no sensitive info).

## Rollback

Two equivalent mechanisms:

1. **Delete the newer release** on GitHub (web UI or
   `gh release delete <tag>`). Clients will pick the next-highest semver on
   their next poll. This is the preferred mechanism because it also hides
   the bad version from anyone who hasn't pulled it yet.
2. **Re-publish an older version number** as a new release. Because
   semver-max picks the highest version (not the newest publish), an older
   version number will not win; this only works if you also delete the
   newer release.

Binary clients that have already staged the bad version will not roll back
automatically — rollback only affects *future* polls. To force a rollback
on a host that has already applied the bad version, set
`TM_UPDATE_SOURCE` to a URL serving the older manifest, or replace the
binary on disk manually and clear `staged.json`.

## Chicken-and-egg migration

Existing frozen fleets carry a binary that has `DEFAULT_GITHUB_REPO` baked
in as the **private** `California3/task-manager`. That repo returns 404
without a token, so those fleets cannot use the new public repo
(`California3/task-manager-app`) to upgrade themselves — they would first
need to be running a version that already knows the new repo URL.

Resolution: **double-publish for 1-2 versions** during the transition
window. For the next 1-2 Task Manager releases after this repo is live,
the central build host will upload the binary both:

- to the new public repo (`California3/task-manager-app`, tag `tm-<ver>`),
  as specified in this document, and
- to the legacy private repo (`California3/task-manager`,
  tag `tm-<ver>`), so that old clients whose baked-in fallback points at
  the private repo can still upgrade.

The version published to the legacy repo is the **first version that knows
about the new public repo**. Once a fleet is running that version, its
fallback target is the new public repo, and the legacy repo can stop being
updated. The cutoff version and date will be announced in the release
notes of the final double-published version.

Fleets that miss the cutoff window must manually redeploy: replace the
binary on disk and set `TM_GITHUB_REPO=California3/task-manager-app` (via
env or the Settings UI once running) to rejoin the normal update channel.

## Pagination & rate limit notes

- **Pagination**: `per_page=100` returns up to 100 releases across **all
  three channels combined**. A single channel with ~33 releases would
  already exhaust one page. Clients must follow `Link: rel="next"` from
  day one; there is no cap-and-paginate-later path.
- **Rate limit (anonymous, 60/h)**: single host, three channels, ~30min
  poll → ~6 req/h. Fine. ETag 304s do not count.
- **Rate limit (PAT, 5000/h)**: recommended for any fleet >10 hosts
  behind one egress IP. PAT scope `public_repo` (classic) or read-only on
  this one repo (fine-grained) is sufficient; do **not** use the broader
  `repo` scope.
