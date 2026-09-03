# prism-releases

Single source of truth for the latest GA versions of PRISM components.

## versions.json schema

```json
{
  "prism-arke-vscode": "0.3.9",
  "prism-arke": "0.6.0",
  "prism-iris-gateway": "0.8.2"
}
```

Each key matches the GitHub repo name of the component. Values are numeric semver strings (`MAJOR.MINOR.PATCH`).

## Raw URL

```
https://raw.githubusercontent.com/alex-chuong/prism-releases/main/versions.json
```

This URL is fetched by the `prism-arke-vscode` extension at upgrade time to determine whether a newer version is available before downloading the VSIX.

## Updating

Each component's CI workflow updates its own key on every release via the `prism-releases-bot` GitHub App. Do not edit `versions.json` manually except to correct a bad value.

| Component | CI ticket |
|---|---|
| prism-arke-vscode | PRISM-221 |
| prism-arke | PRISM-222 |
| prism-iris-gateway | PRISM-223 |

## GitHub App setup (one-time)

The `prism-releases-bot` GitHub App must be installed on this repo for CI to work. See PRISM-221 for setup instructions.
