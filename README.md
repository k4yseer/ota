# ota — mock of the OrcaSlicer profile OTA / PR-approval scheme

This repo is a **stand-in for [`OrcaSlicer/orcaslicer-profiles`](https://github.com/OrcaSlicer/orcaslicer-profiles)**.
It exists so a (separate) vendor-facing frontend can be built and demoed against a repo we
control, before the real OTA assets are published on `orcaslicer-profiles`.

The frontend never talks to this repo specially — it calls the **GitHub REST API** against a
configurable repo coordinate:

```
{owner}/{repo} = k4yseer/ota                     # now (this mock)
{owner}/{repo} = OrcaSlicer/orcaslicer-profiles  # later (production)
```

Swapping to production is a one-line config change — the endpoints below are identical.

## The two things the frontend reads

### 1. "Has my PR been approved?" → the Pull Request object
Approval is the PR itself, read from the GitHub API:

| Question | Endpoint | Field |
| --- | --- | --- |
| Is it merged / still open? | `GET /repos/{owner}/{repo}/pulls/{n}` | `state` (`open`/`closed`), `merged` (bool), `merged_at` |
| Has a reviewer approved it? | `GET /repos/{owner}/{repo}/pulls/{n}/reviews` | a review with `state: "APPROVED"` |
| List a vendor's submissions | `GET /repos/{owner}/{repo}/pulls?state=all` | filter by the title convention below |

**Approval signal:** treat **`merged` / `merged_at` set** as "approved & accepted." Merge is the true
OTA trigger (merge → release → distributed to users).

### 2. "Where do I download the OTA files?" → GitHub Releases
The built profile zips are **release assets**, one release tag per OrcaSlicer version:

```
GET /repos/{owner}/{repo}/releases
  → [ { tag_name: "2.5.1", assets: [ { name: "2.5.1_qidi_2026_07_17.zip", browser_download_url } ... ] },
      { tag_name: "2.5.0", assets: [ ... ] } ]
```

Releases `2.5.0` and `2.5.1` already exist, each with the six vendor bundles attached.

## Conventions (so the frontend can parse a PR)

A PR represents one vendor submitting a profile update. The vendor, version and date are carried by the
**bundle filename** the PR adds — `<version>_<vendor>_<date>.zip` (e.g. `2.5.1_qidi_2026_07_17.zip`) — so
the frontend should parse that rather than a title prefix. The filename is available in two places:

- **PR head branch** = the vendor slug, e.g. `qidi` / `bbl` / `snapmaker` (`GET /pulls/{n}` → `head.ref`).
- **Changed file** = `profiles/<Vendor>/<version>_<vendor>_<date>.zip` (`GET /pulls/{n}/files` → `filename`);
  the auto-generated PR title `Create <version>_<vendor>_<date>.zip` also contains it.

Suggested parse (matches the current demo PRs #1–#3):
`/(\d+\.\d+\.\d+)_([a-z]+)_(\d{4}_\d{2}_\d{2})\.zip/` → `[version, vendor, date]`.

Vendors currently mocked: `BBL`, `Qidi`, `Snapmaker`, `Sfsdf` (a deliberate junk vendor, handy for the
"rejected" state). Each vendor's bundles live under `profiles/<Vendor>/`.

## Verify (same calls the frontend makes)

```bash
curl -s "https://api.github.com/repos/k4yseer/ota/pulls?state=all" \
  | python -c "import sys,json;[print('#%s'%p['number'],p['state'],'merged' if p['merged_at'] else '-',p['title']) for p in json.load(sys.stdin)]"

curl -s "https://api.github.com/repos/k4yseer/ota/releases" \
  | python -c "import sys,json;[print(r['tag_name'],[a['name'] for a in r['assets']]) for r in json.load(sys.stdin)]"
```
