# prism-releases

Single source of truth for the latest GA versions of PRISM components, and for the integrity
metadata of the artifacts those versions refer to.

## versions.json schema

```json
{
  "prism-arke-vscode": "<version>",
  "prism-arke": "<version>",
  "prism-iris-gateway": "<version>",
  "artifacts": {
    "<component>": {
      "version": "<version>",
      "files": [
        { "platform": "<platform>", "name": "<filename>", "sha256": "<hex>" }
      ]
    }
  }
}
```

The actual current values are in [`versions.json`](versions.json) — the example above uses
placeholders to avoid becoming stale.

### Top-level version keys

Each key matches the GitHub repo name of the component. Values are numeric semver strings
(`MAJOR.MINOR.PATCH`) with **no leading `v`**. Consumers reject anything that does not match
`^\d+(\.\d+)+$`, so a prerelease suffix here reads as "unknown version", not as a downgrade.

These three keys are the only thing older consumers read. Nothing may be removed from or
renamed in this set without checking the consumer list below.

### `artifacts`

Per-component metadata for the files published to IBM Box for that release.

| Field | Meaning |
|---|---|
| `version` | The release these files belong to. **Must** equal the component's top-level version key. |
| `files[].platform` | `macos-arm64`, `macos-amd64`, `linux-amd64`, `windows-amd64`, or `any`. |
| `files[].name` | Exact filename as published, including extension. |
| `files[].sha256` | Lowercase hex sha256 of that file, or `null` when not yet published. |

Two rules make this block safe to consume:

1. **`version` is the staleness guard.** A consumer must compare `artifacts.<component>.version`
   against the top-level version key and ignore the whole block when they disagree. That is what
   makes a half-updated file fail closed instead of validating a new download against an old
   hash.
2. **`sha256: null` means "not published", not "no hash needed".** A consumer must treat null as
   unverifiable and say so, never as a pass.

`sha256` is `null` for every file at the time of writing. Populating it is step 3 of the
migration below — see the honest-state note there.

### Artifact filenames

`files[].name` is not free-form. For `prism-arke` it reproduces the goreleaser
`archives[].name_template` in [`prism-arke/.goreleaser.yml`](https://github.com/Apptio-PNE/prism-arke/blob/main/.goreleaser.yml),
which is the authoritative naming contract:

| Platform | Filename |
|---|---|
| macos-arm64 | `arke-v<version>-macos-arm64.tar.gz` |
| macos-amd64 | `arke-v<version>-macos-amd64.tar.gz` |
| linux-amd64 | `arke-v<version>-linux-amd64.tar.gz` |
| windows-amd64 | `arke-v<version>-windows-amd64.zip` |

Note the `v` prefix on the version inside the filename, and `macos` rather than goreleaser's
default `darwin`. Box hosts the goreleaser archives; the `make bundle-*` zips are dev-only and
never uploaded (`packaging/upgrade.sh` says so at the top).

For `prism-arke-vscode` the filename is `prism-arke-vscode-<version>.vsix`.

The `prism-arke-vscode` extension **pins its Downloads glob to these names**. It watches for
`arke-v<version>-<os>-<arch>` derived from the version published here, rather than accepting any
`arke*-<os>-<arch>` file, because it then extracts that archive and runs the installer inside it
under sudo / UAC. If a release ever changes the naming template, the extension's
`pinArchivePrefix()` must change in the same window or its installs will time out.

## Raw URL

```
https://raw.githubusercontent.com/alex-chuong/prism-releases/main/versions.json
```

## Consumers

Everything that reads this file. Update all of them when the repo moves or the schema changes.

| Consumer | What it reads | What breaks if this file is wrong |
|---|---|---|
| `prism-arke-vscode` — dashboard open | both top-level versions | Upgrade banners appear or fail to appear |
| `prism-arke-vscode` — `Arke: Check for Updates` | both top-level versions | Update notifications |
| `prism-arke-vscode` — `Arke: Upgrade Extension` pre-flight | `prism-arke-vscode` | An "already up to date" skip, or a needless download |
| `prism-arke-vscode` — Box install + upgrade flows | `prism-arke` | The pinned Downloads glob. A wrong or stale value makes the install time out |

Every one of those degrades to unknown-version behaviour on fetch failure, by design: an
unreachable version server must never block an install or an upgrade.

## Updating

Each component's CI workflow updates its own key on every release via the `prism-releases-bot`
GitHub App. Do not edit `versions.json` manually except to correct a bad value.

| Component | CI ticket |
|---|---|
| prism-arke-vscode | PRISM-221 (Done) |
| prism-arke | PRISM-222 |
| prism-iris-gateway | PRISM-223 |

### Release procedure for the `artifacts` block

When publishing a `prism-arke` release, the same job that bumps the version key must also rewrite
that component's `artifacts` entry, in one commit:

1. Take the sha256 values from the checksums file goreleaser already produces for the tag —
   `arke-v<version>-checksums.txt`, declared in `.goreleaser.yml` under `checksum.name_template`
   with `algorithm: sha256`. Do not recompute them from a locally rebuilt binary; the value must
   describe the bytes that were actually uploaded to Box.
2. Verify the uploaded Box file matches that hash before publishing it here. A hash copied from a
   release whose asset was replaced during upload is worse than no hash.
3. Set `artifacts.prism-arke.version` to the new version and replace all four `files` entries.
   Never leave a stale hash next to a new version — a consumer that trusts the staleness guard
   will happily validate against it.

## GitHub App

The `prism-releases-bot` GitHub App (App ID: 4813798) is installed on this repo and used by all
component CI workflows to commit version updates. See PRISM-221 for setup details.

## Pending: this repo is a trust root, and signing is how that gets fixed

**This file is what every installed copy of the extension believes about which release it should
be on, and it is hosted outside the org on a personal GitHub account.**

Whoever controls this account can trigger fleet-wide upgrade prompts — which lend legitimacy to a
maliciously delivered artifact — or freeze the published version so known-vulnerable installs stop
being prompted. A departed or compromised account has the same effect. The artifacts themselves
are in IBM Box behind w3id SSO, which is what bounds the impact today: the exposure is the
metadata, not the binaries.

Raised from housekeeping to security posture by the 2026-09-10 review of `prism-arke-vscode`.

**Moving this repo into `Apptio-PNE` is not the fix and is not available.** An installed extension
holds no credentials, so this file has to be anonymously readable; `raw.githubusercontent.com` on a
private repo requires a token, and the org does not permit public repos. A GitHub transfer would
not have been sufficient anyway — installed builds have the old URL compiled in, and a new repo
created at an old path takes priority over GitHub's redirect.

The plan is to stop trusting the host rather than move it:

1. **Sign these bytes.** An Ed25519 detached signature published beside this file as
   `versions.json.sig`, verified by the extension *before* it parses the body. The private key
   lives as an `Apptio-PNE` organisation secret scoped to the two publishing workflows
   (`prism-arke-vscode` and `prism-arke`, both org-owned and private); the public key is compiled
   into the extension. The org holds the authority, this repo holds only bytes, and **no public
   org repo is needed**.
2. **Add `signed_at` / `expires_at`** to this file so the signature covers freshness. Signing
   alone does not stop a hostile host from serving a stale-but-genuine file; an expiry window lets
   the extension say "version information is stale" instead of "you are up to date".
3. **Populate `artifacts[].sha256`** per the release procedure above, and only then wire the
   extension to verify downloads against the now-signed manifest.

Ordering matters: publishing hashes before the signature is in force only moves the question from
"who controls the version" to "who controls the hashes". That is why every `sha256` here is `null`
and the extension ships no verifier — a check that cannot fail is worse than a documented gap.

Custody of this repo should also move off an individual's personal account to a team-held machine
account whose credentials live in the team secret store, and **the repo must never be deleted and
its name never freed** — installed builds poll this exact URL indefinitely, and a released name is
claimable by anyone.

Full plan, threat model, and acceptance criteria:
`prism-arke-vscode/docs/TO-ADDRESS.md` item 8.
