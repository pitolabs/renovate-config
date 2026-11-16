# renovate-config

The Renovate presets every pitolabs repository extends, as
[`base-repo-standard`](https://github.com/pitolabs/builder-skills/tree/main/skills/base-repo-standard) asks. Public
because a public repository cannot read a private preset.

| File | Extended as | Sets |
|---|---|---|
| [`default.json`](default.json) | `local>pitolabs/renovate-config` | Renovate's `config:best-practices`, plus the rules below |
| [`automerge.json`](automerge.json) | `local>pitolabs/renovate-config:automerge` | part of the standard: every update but majors and re-pushed actions or release assets (minor, patch, pin, Docker digest, 0.x included) in one PR, `automerged updates`, and the weekly lockfile refresh, merge on green CI |

## What the default sets, and why

- **`config:best-practices`**: actions and Docker images pinned by digest, config migration PRs, dev dependencies
  pinned, abandoned packages flagged, npm releases held three days, lockfiles refreshed weekly.
- **PyPI releases wait three days too**, like npm: time for a malicious or broken release to be found and pulled
  before Renovate proposes it.
  Everything else waits 12 hours, except the weekly lockfile refresh: Renovate has no release dates for what a refresh
  pulls in, so any age gate kept it pending for ever (`renovate/stability-days`,
  [renovate#45236](https://github.com/renovatebot/renovate/issues/45236)); CI and osv-scanner are its guard. Digest
  updates have the same missing date at the branch stage, so they get `minimumReleaseAgeBehaviour:
  timestamp-optional`; their age is still checked against the tag's push time when Renovate looks them up.
- **Automerge is part of the standard**, through `:automerge`: minor, patch, pin, digest-pin and Docker digest
  updates, 0.x dependencies like any other, and the weekly lockfile refresh, merge once CI is green, so updates do
  not pile up. It needs CI that runs the repository's tests on pull requests (the base-repo-standard audit checks
  both); those tests are the only gate before a merge. Majors, and re-pushed actions or release assets (below), wait
  for a human. The updates that merge themselves go in one PR per repository, `automerged updates`: each PR costs a
  CI run, and another each time it is brought up to date with `main`. A group automerges only when every update in it
  does and its CI is green, so one update that breaks the build holds the whole group until someone fixes or excludes
  it; a repository's own `groupName` rules (later in its `renovate.json`) win.
- **Monday and Tuesday only (UTC)**: `schedule` and `lockFileMaintenance.schedule` are `* * * * 1-2`, and
  `updateNotScheduled: false` makes Renovate leave open PRs alone on the other days: no update with `main`, and no
  merge either (Renovate's default keeps bringing them up to date, so every push to `main` re-ran CI on every open
  automerge PR; in November 2026 that was over half the CI minutes of the private repositories, which pay for
  Actions). Two days, because Renovate merges one PR per run and the next PR must then be brought up to date and pass
  CI before it merges. Outside the window, only a PR's own rebase checkbox makes Renovate update and merge that PR.
- **Majors** are labelled `major-update` and wait for a human.
- **Digest updates** (same tag, new content) of **Docker images** merge with the rest: official images are rebuilt
  under the same tag for base-image patches all the time. Of anything else (actions, release assets) they mean the
  upstream re-pushed a tag, as attackers did with `trivy-action`'s tags in March 2026: labelled `upstream-repush`,
  they never automerge.
- **Vulnerability alerts** (from Dependabot alerts, which must be on in the repository) open a PR labelled `security`
  at once, outside the schedule and hourly limit.
- `prHourlyLimit: 0` (the default of 2 held PRs back for hours), `platformAutomerge: false` and `ignoreTests: false`:
  Renovate itself merges, only after the checks pass. It merges one PR per branch per run; the Mend-hosted app runs
  every few hours. To run it now, tick the checkbox at the bottom of the Dependency Dashboard issue.
- `mode: full` takes the repository out of Mend's Silent mode (the default when the app is installed for all
  repositories). If no Dependency Dashboard issue appears after two runs, turn it off by hand: developer.mend.io → the
  repository → Settings → Dependencies → Silent mode off, Automated PRs on.

Repository-specific managers, groups and labels stay in that repository's `renovate.json`.

## Adopt it

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>pitolabs/renovate-config", "local>pitolabs/renovate-config:automerge"]
}
```

## Change it

`main` takes changes only through a pull request whose `validate` (Renovate's config validator) and `base` checks
pass, and never a direct push, a force push or a deletion; there is no bypass. A change here reaches every repository
on its next Renovate run. This repository follows base-repo-standard and public-repo-standard (`sh scripts/setup.sh`
turns on the hooks and installs the pinned tools).

Merge a pull request locally once its checks pass: push its tested head to `main`, a fast-forward. GitHub records the
pull request as merged, and the commits keep the noreply identity of your git config:

```bash
git fetch origin && git push origin origin/<branch>:main   # the branch must be up to date with main
```

Do not merge on GitHub (merge, squash or rebase) until **Keep my email addresses private** is on
(github.com/settings/emails): GitHub makes that commit and records the merger with this setting, not with your git
config, so with it off your primary address becomes the merge's author or committer, in a public history, and the
`privacy` job refuses it from then on. Renovate's own merges carry its bot identity.
