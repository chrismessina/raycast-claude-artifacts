# Contributing to Claude Artifacts

Thanks for helping. This guide covers running Claude Artifacts locally and sending a change.

## Where changes go

Claude Artifacts is published in the [Raycast Store](https://www.raycast.com/chrismessina/claude-artifacts).
Open pull requests against [`raycast/extensions`](https://github.com/raycast/extensions/tree/main/extensions/claude-artifacts),
not this repository: Raycast reviews every change there, and merged changes sync back here
automatically. Open [issues](https://github.com/chrismessina/raycast-claude-artifacts/issues) here or in
`raycast/extensions`, whichever you prefer.

## Run it locally

You need [Raycast](https://www.raycast.com/) and Node.js 22 or later.

1. In Raycast, find Claude Artifacts in the Store and choose **Fork Extension** from the action panel.
   Raycast checks out only this extension's folder from `raycast/extensions`.
2. In that folder, run `npm ci`, then `npm run dev`. Raycast opens the development version, and
   saving a file reloads it.

To see real data, install the recording hook as the README's Setup section describes (it needs
[Claude Code](https://code.claude.com), `jq` and `perl`), or write rows to `~/.claude/artifacts.json`
by hand in the README's Index format.

## Scripts

| Script | What it does |
| --- | --- |
| `npm run dev` | Run the extension in Raycast and reload on save |
| `npm run build` | Build the distribution bundle |
| `npm run lint` | Run Raycast's ESLint and Prettier checks |
| `npm run fix-lint` | Fix what the linter can fix |

## How it works

There is no artifacts API for individual users, so the extension reads a local file.

- `scripts/record-artifact.sh` is a Claude Code `PostToolUse` hook on the `Artifact` tool. It
  appends each publish to `~/.claude/artifacts.json` under a `flock(2)` lock, and every failure
  path exits 0 so it never fails a Claude Code turn. It lives in `scripts/`, not the bundle, so a
  user can read it before installing it.
- `scripts/probe-artifact-hook.sh` captures the hook payload for checking. The `Artifact` tool's
  response shape is undocumented; `docs/hook-payload.md` records the observed payload and the
  locking history.
- `src/utils/index-file.ts` reads and parses the index, forgivingly: it skips bad rows, de-dupes by
  `id`, and sorts undated rows last. `src/types/artifact.ts` is the row type.
- `src/search-artifacts.tsx` is **Search Artifacts**. Pins live in LocalStorage
  (`src/actions/pin.tsx`), never in the index, because the hook owns that file.
- `src/doctor.tsx` is **Run Doctor**, with its checks in `src/utils/doctor.ts` and hook-registration
  reading in `src/utils/hook-status.ts`. **Backfill** scans the Claude Code transcripts under
  `~/.claude/projects/` (`src/utils/transcripts.ts`) and only appends rows the index lacks.
- The `src/utils/` modules import `@chrismessina/raycast-kit` by subpath (`/errors`, `/plural`) so
  they load in plain Node, without `@raycast/api`.

## Help port it to Windows

Claude Artifacts is macOS-only, and not for any reason in the UI: the hook that fills the index is
`bash` plus `jq`, and Claude Code on Windows would need a PowerShell equivalent. Chris doesn't run
Windows, so he can't build or verify the port, and shipping an untested platform claim is worse than
declaring macOS-only. If you develop on Windows, this is the most useful contribution the extension
could get. Here is what it would take:

1. **`scripts/record-artifact.ps1`** — a PowerShell port of the hook. The logic is small but the hard parts are not the syntax: it needs the same **upsert by `id`** (republishing reuses the URL, so appending duplicates), the same **atomic write**, and the same **kernel-backed cross-process lock**. That last one is where the bash version lost two rounds: a lockfile-plus-age-based-reaper design silently drops rows under burst publishing and cannot be fixed by tuning the threshold — see [`docs/hook-payload.md`](https://github.com/chrismessina/raycast-claude-artifacts/blob/main/docs/hook-payload.md). On Windows the equivalent is an exclusive `FileStream` (`[System.IO.FileShare]::None`), which the OS releases on process death; do not reimplement the reaper.
2. **`platforms: ["macOS", "Windows"]`** in `package.json`, plus restoring the platform-explicit form on the one custom shortcut (`{ macOS: …, Windows: … }` on "Reveal Index File" — a bare `cmd` binding does not exist on Windows).
3. **Verifying `$HOME` expansion**, or substituting `$env:USERPROFILE`, in the hook's `command` field on Windows. This is unverified there; on macOS it works (see [`docs/hook-payload.md`](https://github.com/chrismessina/raycast-claude-artifacts/blob/main/docs/hook-payload.md)).
4. **Confirming the payload shape is identical** on Windows — run [`scripts/probe-artifact-hook.sh`](https://github.com/chrismessina/raycast-claude-artifacts/blob/main/scripts/probe-artifact-hook.sh)'s PowerShell equivalent and compare against the captured macOS payload.

There's a plausible better path than a second script: a **"Setup Hook" command** inside the extension that writes the correct hook for the host platform, using [`runPowerShellScript`](https://developers.raycast.com/utilities/functions/runpowershellscript) on Windows. That would improve the setup story on macOS too, which today is "copy a file and hand-edit JSON."

Attach a capture from the probe (step 4) to your issue or pull request. It answers whether the
payload shape holds on Windows, which is the one fact the port depends on and the one Chris can't
check himself.

## Before you open a pull request

- `npm run build`, `npx tsc --noEmit` and `npm run lint` all pass.
- You tried the change in Raycast, including its error and empty states.
- `CHANGELOG.md` has a new entry at the top, dated `{PR_MERGE_DATE}`. Raycast fills in the date
  when the pull request merges.
- New or changed screenshots go in `metadata/`, taken with Raycast's Window Capture.
