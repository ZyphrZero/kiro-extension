# Kiro Agent Extension Local Patch

This repository contains a locally patched Kiro agent extension bundle. The `extension.js` file is not official Kiro source code; it is a patched local bundle intended to reduce Windows write failures, streamed write aborts, transient model-service failures, and noisy extension-host logs during Kiro spec/task execution.

> Kiro updates may overwrite this patch. Always back up the original `extension.js` before replacing it.

Chinese notes: [README.md](README.md)

Detailed technical notes: [docs/TECHNICAL_DETAILS_EN.md](docs/TECHNICAL_DETAILS_EN.md) / [中文](docs/TECHNICAL_DETAILS.md)

## Fixed Issues

This patch is intended for Windows Kiro users who see:

- Metadata write failures: `EPERM`, `EBUSY`, `EACCES`
- File write aborts: `Operation was aborted by user or system`, `Stream error`
- Intermittent spec/task failures: `Failed to invoke Spec Task Execution`
- Immediate failures during model high load, throttling, or transient network jitter
- Long no-op shell waits such as `Start-Sleep -Seconds ...; Write-Output "ok"` keeping tasks occupied
- Repeated extension-host warning: `The decoration is empty`
- Duplicated `[Steering] ExistingFiles` entries
- Stale diagnostics causing code-problems formatting to fail

If your Kiro installation does not show these issues, replacing the official bundle is not recommended.

## Files

| File | Description |
| --- | --- |
| `extension.js` | Patched Kiro agent extension bundle |
| `README.md` | Chinese quick guide |
| `README_EN.md` | English quick guide |
| `docs/TECHNICAL_DETAILS.md` | Chinese technical details |
| `docs/TECHNICAL_DETAILS_EN.md` | English technical details |

## Quick Install

Close Kiro first, then run the commands below. Adjust the path to match your Kiro installation:

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"

# 1. Back up the original bundle
Copy-Item $target "$target.original-backup"

# 2. Check the patched bundle syntax
node --check .\extension.js

# 3. Replace the installed bundle
Copy-Item .\extension.js $target -Force
```

Reload Kiro after replacement:

- Run `Developer: Reload Window` in Kiro
- Or fully quit and restart Kiro

If the copy command fails with a permission error, fully close Kiro first. If it still fails, run PowerShell as administrator.

## Verify

Run a quick post-install check:

```powershell
node --check "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"

Select-String -Path "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js" -Pattern "KIRO_STREAM_RETRY_MAX_ATTEMPTS","KIRO_STREAM_RETRY_FOREVER","KIRO_STREAM_RETRY_BACKOFF_MS"
```

Expected results:

- `node --check` reports no syntax errors.
- The `KIRO_STREAM_RETRY_*` constants are found.
- After Kiro reloads, logs no longer add new `The decoration is empty` warnings.
- Metadata writes no longer add new `EPERM`, `EACCES`, or `EBUSY` failures.
- High-load, throttling, and transient network failures are retried before being surfaced.

## Roll Back

If the patch is incompatible with your Kiro version, restore the backup and restart Kiro:

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
Copy-Item "$target.original-backup" $target -Force
```

## Patch Summary

| Area | Patch behavior |
| --- | --- |
| Metadata writes | Retries transient Windows busy errors; falls back to direct target writes when needed |
| Write tools | Encourages smaller `fs_write` / `fs_append` chunks to avoid oversized streamed arguments |
| `tasks.md` generation | Avoids one-shot full-file generation; uses initial `fs_write` plus later `fs_append` calls |
| FileDecoration | Returns `undefined` when there is no decoration, avoiding empty-decoration warnings |
| Model stream errors | Retries high-load, throttling, and transient network errors with bounded backoff |
| Shell no-op waits | Blocks long `Start-Sleep` plus `Write-Output "ok"` / `"retry"` no-op waits |
| ExistingFiles | Deduplicates `getWorkspaceFiles()` results by path |
| Code problems | Skips invalid or out-of-bounds diagnostics instead of failing the whole formatting pass |

Default retry configuration in `extension.js`:

```js
KIRO_STREAM_RETRY_MAX_ATTEMPTS = 5;
KIRO_STREAM_RETRY_FOREVER = false;
KIRO_STREAM_RETRY_BACKOFF_MS = [1500, 4e3, 9e3, 16e3, 25e3];
```

Infinite retry is not recommended by default. During prolonged service overload, it can keep a task occupied indefinitely.

## Known Limits

- Does not fix historical state warnings: `Cannot restore file ... original checkpoint missing for append action`
- Does not fix Kiro bundle metadata warnings: `No bundle location found for extension kiro.kiroAgent`
- Does not retry non-transient failures: authentication errors, usage limits, oversized prompts, user cancellation, parameter validation failures
- Kiro updates may overwrite the patched bundle

## Maintenance

- Back up the official `extension.js` before every replacement.
- Run `node --check` after every bundle edit.
- Check whether Kiro updates have overwritten the patch.
- Only use `extension.js` files from sources you trust.

## Community

Discussion and feedback are welcome at [linux.do](https://linux.do/).

---

<div align="center">
  <p>Made with ❤️ by <a href="https://github.com/ZyphrZero">ZyphrZero</a></p>
</div>
