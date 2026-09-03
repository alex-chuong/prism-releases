# prism-releases

Single source of truth for the latest GA versions of PRISM components.

## versions.json schema

```json
{
  "prism-arke-vscode": "<version>",
  "prism-arke": "<version>",
  "prism-iris-gateway": "<version>"
}
```

Each key matches the GitHub repo name of the component. Values are numeric semver strings (`MAJOR.MINOR.PATCH`). The actual current values are in [`versions.json`](versions.json) — the example above uses placeholders to avoid becoming stale.

## Raw URL

```
https://raw.githubusercontent.com/alex-chuong/prism-releases/main/versions.json
```

This URL is fetched by the `prism-arke-vscode` extension at upgrade time to determine whether a newer version is available before downloading the VSIX.

## Updating

Each component's CI workflow updates its own key on every release via the `prism-releases-bot` GitHub App. Do not edit `versions.json` manually except to correct a bad value.

| Component | CI ticket |
|---|---|
| prism-arke-vscode | PRISM-221 (Done) |
| prism-arke | PRISM-222 |
| prism-iris-gateway | PRISM-223 |

## GitHub App

The `prism-releases-bot` GitHub App (App ID: 4813798) is installed on this repo and used by all component CI workflows to commit version updates. See PRISM-221 for setup details.
