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

| Channel      | Tag pattern                                  | Assets                                                                              |
|--------------|----------------------------------------------|-------------------------------------------------------------------------------------|
| TM binary    | `tm-v<ver>`                                  | `task-manager-<ver>` (no extension) + `task-manager-<ver>.sha256`                   |
| Runtime      | `runtime-<plat>-<name>-v<ver>`               | `<name>-<ver>.tar.gz` + `<name>-<ver>.tar.gz.sha256`                                |
| Plugin       | `plugin__<plat>__<id>__v<ver>` (see note)    | `<id>-<ver>.tar.gz` + `<id>-<ver>.tar.gz.sha256`                                    |

### Latest semantics

Clients pick the latest release for a channel by:

1. listing releases (following `Link: rel="next"` pagination, not capping at 100),
2. filtering by the channel's tag prefix,
3. **excluding** prereleases and drafts,
4. stripping the leading `v` and parsing the trailing `<ver>` as a (lenient)
   semver; tags whose `<ver>` fails to parse are skipped with a log line,
5. selecting the maximum semver.

This means **rollback is delete-the-newer-release**: removing a newer
`tm-v*` release from GitHub causes every client to fall back to the
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

Plugin ids historically may contain `-`, which makes the
`plugin-<plat>-<id>-v<ver>` scheme ambiguous (e.g.
`plugin-linux-x64-my-cool-tool-v1.0.0` cannot be unambiguously split into
`plat=linux-x64`, `id=my-cool-tool`). To resolve this:

- **Tag separator is double underscore (`__`)**:
  `plugin__<plat>__<id>__v<ver>`.
- `<plat>` and `<id>` are still constrained to `[A-Za-z0-9._-]+` for
  filesystem safety, but the `__` separator is reserved and must not appear
  inside `<plat>` or `<id>`.
- Clients match the channel by prefix `plugin__` and recover `<plat>` /
  `<id>` by splitting on `__`; they do not rely on the asset filename
  alone.

Older `plugin-<plat>-<id>-v<ver>` tags (if any exist) are tolerated by
prefix matching but not produced anymore.

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
  scope). Publication is gated by `TM_GITHUB_PUBLISH=1`.
