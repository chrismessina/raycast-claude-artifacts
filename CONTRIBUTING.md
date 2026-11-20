# Contributing to Claude Artifacts

Thanks for helping. This guide covers running Claude Artifacts locally and sending a change.

## Where changes go

Claude Artifacts is published in the [Raycast Store](https://www.raycast.com/chrismessina/claude-artifacts).
Open pull requests against [`raycast/extensions`](https://github.com/raycast/extensions/tree/main/extensions/claude-artifacts),
not this repository: Raycast reviews every change there, and merged changes come back here in a
sync pull request. Open [issues](https://github.com/chrismessina/raycast-claude-artifacts/issues) here or in
`raycast/extensions`, whichever you prefer.

## Run it locally

You need [Raycast](https://www.raycast.com/) and Node.js 22.22.2 or later.

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
| `npm run publish` | Open or update the pull request in `raycast/extensions` (`ray publish`) |
| `prepublishOnly` | Stops `npm publish`: the extension goes to the Raycast Store, never to the npm registry |

## How it works

There is no artifacts API for individual users, so the extension reads a local file.

- `scripts/record-artifact.sh` is a Claude Code `PostToolUse` hook on the `Artifact` tool. It
  upserts each publish into `~/.claude/artifacts.json` by `id` under a `flock(2)` lock, and every failure
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

Claude Artifacts is macOS-only because of its shell tooling, not its UI. The hook that fills the
index is `bash` plus `jq`, Run Doctor's checks and Backfill run through `/bin/sh` with `jq` and
`perl`, and Claude Code on Windows would need PowerShell equivalents of all of them. Chris doesn't run
Windows, so he can't build or verify the port, and shipping an untested platform claim is worse than
declaring macOS-only. If you develop on Windows, this is the most useful contribution the extension
could get. Here is what it would take:

1. **`scripts/record-artifact.ps1`**, a PowerShell port of the hook. The logic is small, and the hard parts are not the syntax. It needs the same **upsert by `id`** (republishing reuses the URL, so appending duplicates), the same **atomic write**, and the same **kernel-backed cross-process lock**. The lock is where the bash version went wrong: a lockfile-plus-age-based-reaper design silently drops rows under burst publishing, cannot be fixed by tuning the threshold, and took two rounds of green tests to see why. See [`docs/hook-payload.md`](https://github.com/chrismessina/raycast-claude-artifacts/blob/main/docs/hook-payload.md). On Windows the equivalent is an exclusive `FileStream` (`[System.IO.FileShare]::None`), which the OS releases on process death; do not reimplement the reaper.
2. **`platforms: ["macOS", "Windows"]`** in `package.json`, plus the platform-explicit form (`{ macOS: …, Windows: … }`) on both custom shortcuts, "Reveal Index File" in `src/search-artifacts.tsx` and Backfill's `BACKFILL_SHORTCUT` in `src/doctor.tsx`, because a bare `cmd` binding does not exist on Windows.
3. **Verifying `$HOME` expansion**, or substituting `$env:USERPROFILE`, in the hook's `command` field on Windows. This is unverified there; on macOS it works (see [`docs/hook-payload.md`](https://github.com/chrismessina/raycast-claude-artifacts/blob/main/docs/hook-payload.md)).
4. **Confirming the payload shape is identical** on Windows. Run [`scripts/probe-artifact-hook.sh`](https://github.com/chrismessina/raycast-claude-artifacts/blob/main/scripts/probe-artifact-hook.sh)'s PowerShell equivalent and compare against the captured macOS payload.
5. **Run Doctor and Backfill.** `src/utils/doctor.ts` runs the hook check through `/bin/sh`, takes Backfill's lock with `perl` and merges with `jq`; `src/utils/hook-status.ts` reads whether `record-artifact.sh` is registered and supplies the setup and repair prompts that have Claude Code install it. Each needs a Windows path, and Backfill's lock must be the same one the Windows hook takes, or a Backfill run and a hook write can drop each other's rows.

A plausible better path than a second script is a **"Setup Hook" command** inside the extension that writes the correct hook for the host platform, using [`runPowerShellScript`](https://developers.raycast.com/utilities/functions/runpowershellscript) on Windows. That would also improve setup on macOS, which today is "copy a file and hand-edit JSON."

Attach a capture from the probe (step 4) to your issue or pull request. It answers whether the
payload shape holds on Windows, which is the one fact the port depends on and the one Chris can't
check himself.

## Before you open a pull request

- `npm run build`, `npx tsc --noEmit` and `npm run lint` all pass.
- You tried the change in Raycast, including its error and empty states.
- `CHANGELOG.md` has a new entry at the top, dated `{PR_MERGE_DATE}`. Raycast fills in the date
  when the pull request merges.
- New or changed screenshots go in `metadata/`, taken with Raycast's Window Capture.
