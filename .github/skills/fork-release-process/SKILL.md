---
name: fork-release-process
description: 'Merge upstream PRs into this fork, cut a HACS-installable release, and contribute clean PRs back upstream. Use when: HACS zip_release, hacs.json, bump manifest version, merge develop into master, release new version, cherry-pick clean commit for upstream PR, git archive zip, gh release create.'
---

# Fork release process (homeassistant-stellantis-vehicles)

## HACS only installs from a Release zip

`hacs.json` has `"zip_release": true, "filename": "stellantis_vehicles.zip"` — HACS will
**not** install from a plain branch/commit download. It fetches
`https://github.com/<owner>/<repo>/releases/download/<tag>/stellantis_vehicles.zip`. If no
matching release+asset exists, HACS fails with a 404, even though the code itself is fine.

## Branch model

- `develop` — feature work, mirrors upstream's default branch.
- `master` — production; only merge `develop` → `master` right before cutting a release
  (matches upstream's own convention — check `git log --oneline master..develop` and
  `develop..master` first so you know exactly what a merge will bring in).

**Mistake made once, don't repeat it:** cutting a release from the wrong branch tip (e.g.
`master`, which didn't have a feature that only existed on `develop`) silently ships an
incomplete build. Before tagging, diff the branch you're releasing from against the other
branch to confirm it actually contains everything intended.

**Second mistake made once, don't repeat it either:** always run `git status` (and `git
pull --ff-only` if behind) before `git add -A` in this repo. `README.md`'s installs badge
and `manifest.json`'s version get bumped by processes outside this workspace (manual
releases, upstream automation); a stale local working tree can silently regress them
backwards if swept up by an unrelated commit.

## Version scheme

`custom_components/stellantis_vehicles/const.py` parses `manifest.json`'s `"version"` as
three numeric dot-separated parts (an optional `-beta.N` suffix is stripped first) — don't
use a non-numeric suffix like `-custom`, it breaks `INTEGRATION_VERSION` parsing. Check
existing tags before picking the next number:
```bash
git tag | grep -E '^2026\.[0-9]+\.'
```

## Cutting a release

```bash
# on the branch/commit you're releasing from, after bumping manifest.json's "version"
git add -A && git commit -m "Bump version to <X.Y.Z>"
git tag <X.Y.Z>
git push origin <branch> && git push origin <X.Y.Z>

# build the zip with integration files at the ZIP ROOT (not wrapped in custom_components/…)
git archive --format=zip -o /tmp/archive.zip <X.Y.Z> -- custom_components/stellantis_vehicles
mkdir -p /tmp/zip_check && unzip -q /tmp/archive.zip -d /tmp/zip_check
cd /tmp/zip_check/custom_components/stellantis_vehicles && zip -q -r /tmp/stellantis_vehicles.zip ./

GH_TOKEN=$(gh auth token --user chmielowiec) gh release create <X.Y.Z> /tmp/stellantis_vehicles.zip \
  --repo chmielowiec/homeassistant-stellantis-vehicles --title "<X.Y.Z>" --notes "..." --target <branch>

# verify HACS will actually be able to fetch it
curl -sI -L "https://github.com/chmielowiec/homeassistant-stellantis-vehicles/releases/download/<X.Y.Z>/stellantis_vehicles.zip" | head -2
rm -rf /tmp/archive.zip /tmp/zip_check /tmp/stellantis_vehicles.zip
```
`git archive` (rather than `zip -r` on the working tree) avoids accidentally bundling
`__pycache__`/`.mypy_cache` into the release asset.

If you ever tag/release the wrong commit, delete it before re-doing it correctly:
```bash
GH_TOKEN=$(gh auth token --user chmielowiec) gh release delete <X.Y.Z> --repo chmielowiec/homeassistant-stellantis-vehicles --yes --cleanup-tag
```

## Contributing a clean PR back upstream

Don't send your fork's `develop` as-is — it likely carries version-bump/release commits
upstream doesn't want. Cherry-pick only the real feature commit(s) onto a fresh branch off
upstream:
```bash
git fetch upstream
git checkout -b <feature-branch> upstream/develop
git cherry-pick <feature-commit-sha>
git log --oneline upstream/develop..<feature-branch>   # sanity check: only your commit(s)
git push -u origin <feature-branch>
```
Then open the PR from `<your-fork>:<feature-branch>` → `andreadegiovine:develop`.
