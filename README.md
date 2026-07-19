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
There is **no `submissions.json`** — approval is the PR itself, read from the GitHub API:

| Question | Endpoint | Field |
| --- | --- | --- |
| Is it merged / still open? | `GET /repos/{owner}/{repo}/pulls/{n}` | `state` (`open`/`closed`), `merged` (bool), `merged_at` |
| Has a reviewer approved it? | `GET /repos/{owner}/{repo}/pulls/{n}/reviews` | a review with `state: "APPROVED"` |
| List a vendor's submissions | `GET /repos/{owner}/{repo}/pulls?state=all` | filter by the title convention below |

**Approval signal:** treat **`merged` / `merged_at` set** as "approved & accepted." Merge is the true
OTA trigger (merge → release → distributed to users). See the self-approval note at the bottom for why
we key on merge rather than an `APPROVED` review.

### 2. "Where do I download the OTA files?" → GitHub Releases
The built profile zips are **release assets**, one release tag per OrcaSlicer version:

```
GET /repos/{owner}/{repo}/releases
  → [ { tag_name: "2.5.1", assets: [ { name: "2.5.1_qidi_2026_07_17.zip", browser_download_url } ... ] },
      { tag_name: "2.5.0", assets: [ ... ] } ]
```

Releases `2.5.0` and `2.5.1` already exist, each with the six vendor bundles attached.

## Conventions (so the frontend can parse a PR)

A PR represents one vendor submitting a profile update. Encode vendor + target version in the PR title
and branch name:

- **PR title:** `[<Vendor>] <version> profile update` — e.g. `[Qidi] 2.5.1 profile update`
- **Branch:** `vendor/<vendor>/<version>-<date>` — e.g. `vendor/qidi/2.5.1-2026-07-18`

Vendors currently mocked: `BBL`, `Qidi`, `Snapmaker`, `Sfsdf` (a deliberate junk vendor, handy for the
"rejected" state). Each vendor's bundles live under `profiles/<Vendor>/`.

## Seeding the demo PRs

The frontend needs PRs in different states to render. There are **0 PRs** by default — create at least
these three (requires the [`gh` CLI](https://cli.github.com/) after `gh auth login`, or the GitHub web UI):

```bash
# --- MERGED  → frontend shows "approved / live" ---
git switch -c vendor/qidi/2.5.1-2026-07-18
cp "profiles/Qidi/2.5.0_qidi_2026_07_17.zip" "profiles/Qidi/2.5.1_qidi_2026_07_18.zip"
git add profiles/Qidi/2.5.1_qidi_2026_07_18.zip
git commit -m "Qidi 2.5.1 profile update (2026-07-18)"
git push -u origin vendor/qidi/2.5.1-2026-07-18
gh pr create -t "[Qidi] 2.5.1 profile update" -b "Bump Qidi profiles for OrcaSlicer 2.5.1"
gh pr merge --squash --delete-branch          # <-- this is the "approval"

# --- OPEN    → frontend shows "pending / under review" ---
git switch main
git switch -c vendor/snapmaker/2.5.1-2026-07-18
cp "profiles/Snapmaker/2.5.0_snapmaker_2026_07_15.zip" "profiles/Snapmaker/2.5.1_snapmaker_2026_07_18.zip"
git add profiles/Snapmaker/2.5.1_snapmaker_2026_07_18.zip
git commit -m "Snapmaker 2.5.1 profile update (2026-07-18)"
git push -u origin vendor/snapmaker/2.5.1-2026-07-18
gh pr create -t "[Snapmaker] 2.5.1 profile update" -b "Bump Snapmaker profiles"   # leave open

# --- CLOSED (unmerged) → frontend shows "rejected" ---
git switch main
git switch -c vendor/sfsdf/2.5.1-2026-07-18
cp "profiles/Sfsdf/2.5.0_sfsdf_2026_07_15.zip" "profiles/Sfsdf/2.5.1_sfsdf_2026_07_18.zip"
git add profiles/Sfsdf/2.5.1_sfsdf_2026_07_18.zip
git commit -m "Sfsdf 2.5.1 profile update (2026-07-18)"
git push -u origin vendor/sfsdf/2.5.1-2026-07-18
gh pr create -t "[Sfsdf] 2.5.1 profile update" -b "Junk vendor"
gh pr close                                    # close without merging
```

> Bundle contents are placeholder text; the demo cares about PR **state** + release **assets**, not the
> zip internals. Paths above assume the `2.5.1_*` bundles exist under `profiles/`; adjust names to match
> what's on disk.

## Verify (same calls the frontend makes)

```bash
curl -s "https://api.github.com/repos/k4yseer/ota/pulls?state=all" \
  | python -c "import sys,json;[print('#%s'%p['number'],p['state'],'merged' if p['merged_at'] else '-',p['title']) for p in json.load(sys.stdin)]"

curl -s "https://api.github.com/repos/k4yseer/ota/releases" \
  | python -c "import sys,json;[print(r['tag_name'],[a['name'] for a in r['assets']]) for r in json.load(sys.stdin)]"
```

## Caveat: GitHub blocks self-approval

You **cannot** submit an `APPROVED` review on your own PR with a single account. That's why the frontend
keys on **`merged`** as the approval signal. If you specifically want to demo the pre-merge
`APPROVED` review state, approve the PR from a **second GitHub account or a bot**.
