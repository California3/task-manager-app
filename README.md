# task-manager-app

Public **artifacts-only** repository for the **Task Manager** platform.

This repository **does not contain source code**. It only hosts:

- channel conventions and release process documentation (in git)
- release binaries / tarballs / sha256 sidecars (as **GitHub Release assets**, never committed to the git tree)

The source code lives in the private repository
[`California3/task-manager`](https://github.com/California3/task-manager) and is
**not** mirrored here. This repo is the public distribution point so that
fleets and external installs can pull Task Manager binaries, runtimes, and
plugins without GitHub authentication when the central build host is
unreachable.

See [`CHANNELS.md`](./CHANNELS.md) for the full release/receive protocol,
central-vs-fallback relationship, rollback, and chicken-and-egg migration
notes.

## Three release channels

All channels share one GitHub repo and are distinguished by **tag prefix**.
There is intentionally no top-level `manifest.json` in the git tree — clients
discover the latest release by listing `/releases`, filtering by tag prefix,
and picking the highest semver.

| Channel      | Tag pattern                          | Assets                                                              |
|--------------|--------------------------------------|---------------------------------------------------------------------|
| TM binary    | `tm-<ver>`                           | `task-manager-<ver>` (no extension) + `task-manager-<ver>.sha256`   |
| Runtime      | `runtime-<plat>-<name>-<ver>`        | `<name>-<ver>.tar.gz` + `<name>-<ver>.tar.gz.sha256`                |
| Plugin       | `plugin-<plat>-<id>-<ver>` (see note)| `<id>-<ver>.tar.gz` + `<id>-<ver>.tar.gz.sha256`                    |

`<ver>` is emitted as-is by the publish scripts and used uniformly in tags,
asset filenames, and the `version` field clients compare against. The
convention differs per channel:

- **TM binary**: `<ver>` carries a leading `v` (e.g. `v3.4.4`), read
  verbatim from `dist/task-manager.version`. Tag → `tm-v3.4.4`, asset →
  `task-manager-v3.4.4`.
- **Runtime / Plugin**: `<ver>` is bare (e.g. `2.1.141`, `0.1.8`), with no
  leading `v`. Tag → `runtime-linux-x64-claude-2.1.141`, asset →
  `claude-2.1.141.tar.gz`.

Clients strip an optional leading `v` before `packaging.version.Version`
parses the remainder, so both forms compare correctly.

### Latest semantics

Clients pick the latest release for a channel by:

1. listing releases (following `Link: rel="next"` pagination, not capping at 100),
2. filtering by the channel's tag prefix,
3. **excluding** prereleases and drafts,
4. stripping the leading `v` and parsing the trailing `<ver>` as a (lenient)
   semver; tags whose `<ver>` fails to parse are skipped with a log line,
5. selecting the maximum semver.

This means **rollback is delete-the-newer-release**: removing a newer
`tm-*` release from GitHub causes every client to fall back to the
next-highest semver on its next poll. Re-publishing an older version number
has the same effect.

### sha256 sidecars

Every asset ships with a sibling `<asset>.sha256` sidecar in the same
release. The sidecar is a **bare lowercase hex string** (a single optional
trailing newline / whitespace is tolerated by clients).

Sidecars are **transport integrity checks**, not supply-chain signatures.
They protect against CDN corruption, truncated downloads, and accidental
re-uploads of a mismatched asset. They do **not** protect against an
attacker who can push to this repo (such an attacker can replace both the
asset and its sidecar). Supply-chain signing is future work.

Clients **must** download the sidecar and verify the asset hash before
staging; a missing sidecar is a hard error, not a fallback.

### Plugin tag format (disambiguation)

Plugin ids may contain `-`, which makes a tag like
`plugin-linux-x64-my-cool-tool-1.0.0` ambiguous **if you try to parse
`<plat>` / `<id>` / `<ver>` back out of the tag by splitting on `-`**.
The implementation avoids this entirely:

- The tag is `plugin-<plat>-<id>-<ver>` with ordinary single-`-` separators.
  `<plat>` and `<id>` are still constrained to `[A-Za-z0-9._-]+` for
  filesystem safety.
- Clients **never parse the tag** for `<plat>` / `<id>` / `<ver>`. The
  caller already knows `<plat>` and `<id>` (it's looking up a specific
  plugin), so it builds the prefix `plugin-<plat>-<id>-` and filters
  releases by `tag_name.startswith(prefix)`.
- `<ver>` is recovered from the **asset filename**, not the tag: the asset
  is `<id>-<ver>.tar.gz`, and the client strips the known `<id>-` prefix
  and the `.tar.gz` suffix. Because `<id>` is known, this is unambiguous
  regardless of `-` inside `<id>` or `<ver>`.

Net effect: the `-`-in-id ambiguity never arises, because no component
ever splits the tag on `-`. The tag exists only for human/URL readability
and as a prefix-filter key.

## Authentication

- **Reading** this repo (releases, assets, sidecars) requires **no token**
  because the repo is public. Unauthenticated GitHub API reads are
  rate-limited to 60 requests/hour per IP, which is fine for a single
  host polling three channels every ~30 minutes.
- **Fleet deployments** (many hosts behind one egress IP) should configure
  a GitHub PAT with `public_repo` scope (classic) or read access on this
  repo (fine-grained) and pass it via `TM_GITHUB_TOKEN` to raise the limit
  to 5000 requests/hour. ETag / `If-None-Match` conditional requests are
  used by clients so that 304 responses do not count against quota.
- **Writing** (publishing releases) is only ever done from the central
  build host, which uses `gh` CLI with `GH_TOKEN` (PAT with `public_repo`
  scope). Publication is on by default; set `TM_GITHUB_PUBLISH=0` to skip.

## Prefer-fallback mode

A client-side opt-in, `TM_GITHUB_PREFER` (env var, or Settings →
Updates checkbox, default off). When on, the client skips the central
source entirely and pulls all three channels (binary / runtime / plugin)
directly from this repo. Use cases: testing the fallback path without
taking central down, central known-bad / stale, debugging the GitHub
consume chain. See `CHANNELS.md` for the full truth table and
short-circuit protection.

## Mirror acceleration

GitHub release assets are served from `release-assets.githubusercontent.com`
(Azure blob) which has bad peering from some networks — observed 16 KB/s
direct vs 11.7 MB/s through `ghfast.top` on the same host. `TM_GITHUB_MIRROR`
(env var, or Settings → Updates text field) lets the client race multiple
mirrors and pick the fastest per download.

Built-in candidates (always present): `DIRECT`, `https://ghfast.top`,
`https://gh-proxy.com`. The env var adds extra comma-separated prefixes.
Before each download, all candidates are probed in parallel (512KB ranged
GET, 8s wall-clock); the fastest wins, with fall-through on failure or
sha256-mismatch. The sha256 sidecar is fetched DIRECT from GitHub (not
mirrored), so retry-on-mismatch is safe. See `CHANNELS.md` for the full
design.
