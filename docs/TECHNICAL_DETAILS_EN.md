# Kiro Agent Extension Local Patch Technical Details

This document is for maintainers, troubleshooters, and users who need to modify `extension.js` again. The root `README_EN.md` keeps only the install and summary path; this file records the patch rationale, key implementation behavior, verification steps, and known boundaries.

中文：[TECHNICAL_DETAILS.md](TECHNICAL_DETAILS.md) | Back to: [README_EN.md](../README_EN.md)

## 1. Document Scope

This repository maintains the already bundled Kiro agent extension file:

```text
extension.js
```

It is not the official Kiro TypeScript source code and it is not a rebuildable source project. The current patch is applied directly to compiled and bundled JavaScript, so maintenance should rely on stable search anchors rather than upstream source file paths.

The patch targets several issues observed during Kiro spec/task execution on Windows:

- `taskUpdate` metadata writes fail with `EPERM`, `EBUSY`, or `EACCES`.
- Large streamed `fs_write` / `fs_append` arguments are aborted before completion.
- `tasks.md` generation repeatedly overwrites the whole file and leaves sub-tasks running for too long.
- Model high load, temporary throttling, or transient network disconnects fail tasks immediately.
- Extension host logs frequent empty `FileDecoration` warnings.
- `[Steering] ExistingFiles` grows with duplicate file entries.
- Stale diagnostics point outside the current document and break code-problems formatting.

## 2. Deliverables and Directory Conventions

The primary user-facing deliverables are:

```text
extension.js
README.md
README_EN.md
```

Technical notes live under `docs/`:

```text
docs/TECHNICAL_DETAILS.md
docs/TECHNICAL_DETAILS_EN.md
```

Do not distribute local backups or temporary files.

Users must back up the original installed bundle before replacement:

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
Copy-Item -LiteralPath $target -Destination "$target.original-backup"
```

Kiro updates may change the official bundle, and this patched bundle may not be compatible with every Kiro version. Keeping the original file allows fast rollback.

## 3. Runtime Environment and Install Paths

Typical Kiro install path:

```text
D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js
```

Typical task metadata path:

```text
C:\Users\<USERNAME>\.kiro\tasks\<WORKSPACE_HASH>\<PROVIDER>.meta.json
```

For example:

```text
C:\Users\<USERNAME>\.kiro\tasks\<TASK_DIR>\codex-provider.meta.json
```

The primary target platform is Windows. Some logic is enabled only when `process.platform === "win32"` to avoid changing file-write behavior on other platforms.

## 4. Patch Overview

| Module | Search anchors | Behavior |
| --- | --- | --- |
| Task metadata writes | `writeMetadataFile`, `EPERM`, `EBUSY`, `EACCES` | Retries Windows transient busy errors; falls back to direct target writes when needed |
| Streamed write errors | `handleStreamError`, `The streamed write input was aborted before complete` | Converts aborted write errors into clear small-chunk retry guidance |
| Write tool description and preflight | `fs_write text is too large`, `fs_append text is too large` | 40-line normal guidance, 80-line hard cap |
| `tasks.md` generation prompt | `The model MUST write tasks.md incrementally` | Prevents one-shot generation and repeated full-file overwrites |
| Model stream retry | `KIRO_STREAM_RETRY_MAX_ATTEMPTS`, `getRetryableStreamErrorKind` | Retries high load, throttling, and transient network failures before the first chunk |
| FileDecoration | `AgentActivityFileDecorationProvider`, `provideFileDecoration` | Returns `undefined` when there is no decoration; returns valid decoration when activity exists |
| Workspace file dedupe | `getWorkspaceFiles()`, `read_file`, `read_files` | Deduplicates by `path` before returning |
| Diagnostics formatting | `Caught error formatting code problems` | Skips stale or out-of-range diagnostics |

## 5. Windows Metadata Write Patch

### 5.1 Original Issue

Kiro task state updates repeatedly write metadata files. The original flow is roughly:

1. Write a temporary file.
2. Use `rename(temp, target)` to replace the target metadata file.

On Windows, `rename` is sensitive to file handles held by Defender, indexers, file watchers, Kiro itself, or other processes. It may throw:

```text
EPERM
EBUSY
EACCES
```

These errors are often transient and do not mean the file is permanently unwritable. Without retry and fallback handling, `taskUpdate` may fail immediately.

### 5.2 Current Implementation

Search in the bundle for:

```text
task-metadata-storage
writeMetadataFile
EPERM
EBUSY
EACCES
```

The patch contains helper logic equivalent to:

| Helper | Role |
| --- | --- |
| `P18(error)` | Detects Windows transient busy errors |
| `B11(ms)` | Sleep helper |
| `G11(fn)` | Retries transient busy errors up to 8 times with exponential backoff |
| `K11(target, temp, content)` | Writes temp, retries rename, then falls back to direct target write if rename remains busy |

Current strategy:

1. `writeMetadataFile()` still creates the task metadata directory first.
2. It generates a random-suffix temp file so concurrent writes do not share a fixed temp path.
3. It writes the temp file.
4. It retries `rename(temp, target)` up to 8 times.
5. If `rename` still fails with a Windows busy error, it falls back to writing the target file directly.
6. It attempts to delete the temp file after fallback or failure.
7. Non-busy errors are not swallowed.

### 5.3 Tradeoff

The patch does not add a global lock and does not change the metadata JSON schema. The observed failure is a transient Windows filesystem contention issue, not a metadata business-model issue.

The direct target write is a fallback, not the primary path. The patch preserves atomic rename behavior when possible and only relaxes it when Windows repeatedly blocks the rename.

### 5.4 Verification

After installation, inspect Kiro logs and confirm there are no repeated failures involving:

```text
codex-provider.meta.json
EPERM
EBUSY
EACCES
taskUpdate
```

Syntax check:

```powershell
node --check "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
```

## 6. Streamed Write Abort Handling

### 6.1 Original Issue

`fs_write` / `fs_append` arguments are part of the streamed model output. If the model emits a long `text` argument, the stream may fail before the argument is complete:

```text
The streamed write input was aborted before complete.
Operation was aborted by user or system
Error(s) while editing, aborted
```

The original error was not actionable enough for the model, so it could retry with the same oversized content and fail again.

### 6.2 Current Implementation

Search for:

```text
handleStreamError
The streamed write input was aborted before complete
```

When `handleStreamError()` sees an error message containing `aborted`, and the current tool is:

```text
fs_write
fs_append
str_replace
```

it returns this recovery hint:

```text
The streamed write input was aborted before complete. Retry with smaller chunks: keep fs_write/fs_append text to 40 lines or fewer, and use multiple str_replace calls for large edits.
```

This gives the model a concrete recovery path: smaller writes or multiple `str_replace` calls.

### 6.3 Why Not 25 Lines

A tighter 25-line cap was considered, but testing showed it creates too many tool calls. In `tasks.md` generation, it can cause repeated `fs_write` overwrites and leave tasks running for too long.

The current policy is:

- 40 lines: normal guidance.
- 80 lines: hard preflight cap.

This reduces abort risk without over-fragmenting tool calls.

## 7. `fs_write` / `fs_append` Chunking Strategy

### 7.1 Tool Description Patch

Search for:

```text
RELIABILITY LIMIT
Keep text to
Very large streamed inputs are rejected before execution
```

The `fs_write` and `fs_append` descriptions now state:

- Keep normal single writes at or below 40 lines.
- For large files, use one `fs_write` for the first chunk.
- Use multiple `fs_append` calls for later chunks.
- Prefer multiple `str_replace` calls for large edits.

### 7.2 Streamed Parameter Preview Preflight

Search for:

```text
fs_write text is too large for reliable streaming
fs_append text is too large for reliable streaming
```

During streamed parameter preview, the tool checks the `text` line count:

- If `text` exceeds 80 lines, it throws a recoverable error.
- This happens before the file write executes.
- The goal is not to prevent large files; it forces the model to split large writes into smaller chunks.

### 7.3 Line Count Rules

| Scenario | Recommendation |
| --- | --- |
| Normal `fs_write` / `fs_append` | 40 lines or fewer |
| Absolute cap | 80 lines or fewer |
| New large file | First chunk with `fs_write`, later chunks with `fs_append` |
| Large edits | Multiple `str_replace` calls |
| `tasks.md` generation | One initial `fs_write`, then `fs_append` |

## 8. `tasks.md` Generation Constraints

### 8.1 Original Issue

When the spec task sub-agent generates `tasks.md`, one-shot full-file writes are likely to trigger streamed-argument aborts. If the line limit is too tight, the sub-agent may repeatedly overwrite the entire file with `fs_write`, leaving the task running for too long.

### 8.2 Current Implementation

Search for:

```text
The model MUST write tasks.md incrementally
After the first successful fs_write for tasks.md
```

The task-generation prompt now requires:

- Use only one `fs_write` to create the title and first complete section of `tasks.md`.
- Use `fs_append` for all later sections.
- Keep each `fs_write` / `fs_append` normally at or below 40 lines, with an absolute cap of 80 lines.
- Do not generate the whole `tasks.md` with a single `fs_write` or `fs_append`.
- After the first successful `fs_write`, do not use another `fs_write` to overwrite the whole `tasks.md`.
- Once all required sections exist, call `subagent_response` and stop.

### 8.3 Expected Effect

This reduces two risks:

- Tool input is aborted before a large payload finishes streaming.
- The sub-agent keeps running because it repeatedly overwrites the same file.

## 9. Model High Load, Throttling, and Network Retry

### 9.1 Original Issue

The Kiro model stream can fail before the first chunk with messages such as:

```text
The model you've selected is experiencing a high volume of traffic. Try changing the model and re-running your prompt.
An unexpected error occurred, please retry.
A network error occurred. Please check your connection and try again.
Too many requests, please wait before trying again.
Failed to invoke Spec Task Execution
```

`Failed to invoke Spec Task Execution` is often not a missing sub-agent. It may be the parent tool wrapping repeated high-load, throttling, or transient network failures inside the sub-agent.

### 9.2 Current Implementation

Search for:

```text
KIRO_STREAM_RETRY_MAX_ATTEMPTS
KIRO_STREAM_RETRY_FOREVER
KIRO_STREAM_RETRY_BACKOFF_MS
_streamResponseChunks
getRetryableStreamErrorKind
```

Core constants:

```js
KIRO_STREAM_RETRY_MAX_ATTEMPTS = 5;
KIRO_STREAM_RETRY_FOREVER = false;
KIRO_STREAM_RETRY_BACKOFF_MS = [1500, 4e3, 9e3, 16e3, 25e3];
```

`_streamResponseChunks()` wraps `_streamResponseChunksOnce()`:

1. If the call succeeds and yields chunks, it returns normally.
2. If it throws before the first chunk, `getRetryableStreamErrorKind()` determines whether the error is retryable.
3. Non-retryable errors are thrown immediately.
4. If any chunk has already been emitted, automatic retry is skipped.
5. If the error is retryable and the retry limit has not been reached, it waits using the backoff schedule and retries.
6. If the limit is reached, it preserves the original error path.

### 9.3 Retried Errors

| Category | Examples |
| --- | --- |
| Model high load | `Encountered unexpectedly high load when processing the request, please try again.` |
| Internal Kiro throttling | `retryErrorType === "THROTTLING"` |
| Temporary throttling | `Too many requests, please wait before trying again.` |
| Network error classes | `B14`, `be10` |
| Network error codes | `ECONNRESET`, `ECONNABORTED`, `ECONNREFUSED`, `EHOSTUNREACH`, `ENETUNREACH`, `ENOTFOUND`, `EAI_AGAIN`, `ETIMEDOUT`, `ERR_NETWORK` |
| Network messages | `network error`, `failed to fetch`, `fetch failed`, `socket disconnected` |
| Server requests retry | `please retry`, `please try again`, `temporarily unavailable`, `An unexpected error occurred` |

The matcher also accepts:

```text
please wait beforetrying again
```

This accounts for malformed spacing seen in some copied/logged messages.

### 9.4 Non-Retried Errors

The patch does not retry:

- Authentication failures.
- Kiro access unavailable errors.
- Hourly/daily/weekly/monthly usage limits.
- Overage limits.
- Prompt or context length failures.
- Parameter validation failures.
- User cancellation.
- Any failure after at least one model chunk has already been emitted.

These are usually not transient service jitter. Retrying them wastes requests and can create duplicate side effects.

### 9.5 Why Retry Only Before the First Chunk

Once the model emits a chunk, downstream behavior may already include tool calls, file writes, or visible partial output. Retrying at that point risks:

- Duplicate file writes.
- Duplicate tool execution.
- Repeated or conflicting responses.
- Unclear state about whether the previous attempt had side effects.

Therefore this patch only retries before the first stream chunk.

### 9.6 Infinite Retry Switch

`KIRO_STREAM_RETRY_FOREVER` defaults to `false`.

It is not recommended by default because:

- During prolonged service overload, the task can remain in running state indefinitely.
- Free-account or temporary throttling may last long enough to hide the real state.
- Users may think the task is still making progress.

Users who explicitly want long waiting can change:

```js
KIRO_STREAM_RETRY_FOREVER = true;
```

This ignores `KIRO_STREAM_RETRY_MAX_ATTEMPTS`.

## 10. Empty FileDecoration Fix

### 10.1 Original Issue

The extension host may repeatedly log:

```text
INVALID decoration from extension 'kiro.kiroAgent': Error: The decoration is empty
```

The cause is that `provideFileDecoration()` returns an empty object or an object without display fields.

### 10.2 Current Implementation

Search for:

```text
AgentActivityFileDecorationProvider
provideFileDecoration
Kiro agent activity
```

Current behavior:

- If decoration is not visible, return `undefined`.
- If the file has no agent activity, return `undefined`.
- If activity exists, return a valid object:

```js
{
  badge: "E",
  tooltip: "Kiro agent activity: ...",
  color: new ThemeColor("list.highlightForeground"),
  agentExecutionState: state
}
```

`badge`, `tooltip`, and `color` satisfy the display fields required by VS Code/Kiro `FileDecoration` validation.

## 11. `getWorkspaceFiles()` Deduplication

### 11.1 Original Issue

Kiro collects file references from several places:

- Explicit file documents.
- `read_file` tool calls.
- `read_files` tool calls.
- Historical context.

The previous `removeDuplicateFiles()` handled document-type entries only and did not deduplicate tool-use paths, so logs could grow with duplicates:

```text
[Steering] ExistingFiles: ...
```

### 11.2 Current Implementation

Search for:

```text
getWorkspaceFiles()
read_file
read_files
InvalidFilePathInReadFile
```

`getWorkspaceFiles()` collects files using the original logic, then filters by `path` with a `Set`:

- Keeps the first occurrence.
- Drops later duplicates.
- Does not alter the path string.
- Does not attempt case normalization, realpath resolution, or symlink resolution.

### 11.3 Boundary

This is conservative string deduplication. It does not treat these as the same file:

```text
C:\Project\File.ts
c:\project\file.ts
..\project\file.ts
```

This avoids introducing path-normalization side effects in a bundled patch.

## 12. Stale Diagnostics Skipping

### 12.1 Original Issue

After quick file edits, old diagnostics may point to lines that no longer exist. The original formatter directly accesses the text line for `diagnostic.range.start.line`; if it is out of bounds, the whole code-problems formatting pass may fail:

```text
Caught error formatting code problems, skipping for now
```

### 12.2 Current Implementation

Search for:

```text
Caught error formatting code problems
diagnostic.range.start.line
diagnostic.range.end.line
```

Before processing each diagnostic, the current logic checks:

- `range` must exist.
- `start.line` and `end.line` must not be negative.
- `start.line` and `end.line` must be smaller than the current document line count.

Invalid diagnostics are skipped instead of failing the whole code-problems block.

## 13. Installation Verification Flow

### 13.1 Before Replacement

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
Copy-Item -LiteralPath $target -Destination "$target.original-backup"
node --check ".\extension.js"
```

### 13.2 Replacement

```powershell
Copy-Item -LiteralPath ".\extension.js" -Destination $target -Force
```

If this fails:

1. Fully quit Kiro.
2. Confirm no Kiro process remains.
3. Run the copy again.
4. If it still fails, use administrator PowerShell.

### 13.3 After Replacement

```powershell
node --check $target
Select-String -Path $target -Pattern "KIRO_STREAM_RETRY_MAX_ATTEMPTS","KIRO_STREAM_RETRY_FOREVER","KIRO_STREAM_RETRY_BACKOFF_MS"
```

Then run:

- `Developer: Reload Window`
- Or fully restart Kiro.

### 13.4 Runtime Observation Points

After startup, inspect Kiro logs:

```text
%APPDATA%\Kiro\logs
```

Confirm:

- Empty decoration warnings no longer appear continuously.
- Metadata writes no longer fail directly on `EPERM`, `EBUSY`, or `EACCES`.
- High load, throttling, and transient network disconnects produce retry debug logs before failing.
- `Failed to invoke Spec Task Execution` appears less frequently.
- `[Steering] ExistingFiles` no longer contains large repeated path lists.

## 14. Troubleshooting

### 14.1 `Too many requests` Still Appears

If the service is still throttling after 5 retries, the error will still surface. This is expected. The default patch is not infinite retry.

Check:

```powershell
Select-String -Path $target -Pattern "retryErrorType === `"THROTTLING`"","Too many requests"
```

For longer waiting, adjust:

```js
KIRO_STREAM_RETRY_MAX_ATTEMPTS = 10;
```

Do not enable this by default unless you understand the tradeoff:

```js
KIRO_STREAM_RETRY_FOREVER = true;
```

### 14.2 Usage-Limit Errors Still Appear

Examples:

```text
You've reached your hourly usage limit.
You've reached your daily usage limit.
You've reached your monthly usage limit.
Overage limit reached
```

These are not transient network or service jitter and are not retried. Wait for quota recovery or change account/plan.

### 14.3 `Failed to invoke Spec Task Execution` Still Appears

This message is a wrapper from the parent tool. Inspect the earlier real error:

- High load.
- Temporary throttling.
- Network disconnect.
- `tasks.md` write failure inside the sub-agent.
- Permission or authentication failure.

If the preceding error is high load, throttling, or transient network failure, the patch retries first. If retries are exhausted, the parent may still report invoke failure.

### 14.4 File Writes Still Abort

Check whether the model is still trying oversized writes:

- Does one `fs_write` / `fs_append` exceed 80 lines?
- Is the model repeatedly using `fs_write` on the same `tasks.md`?
- Should the edit be split into multiple `str_replace` calls?

### 14.5 Patch Disappears After Kiro Update

Kiro updates may overwrite the installed `extension.js`. After updating:

1. Check whether the installed `extension.js` still contains `KIRO_STREAM_RETRY_MAX_ATTEMPTS`.
2. If it does not, the patch was overwritten.
3. Back up the new official bundle.
4. Replace it with the patched bundle again.

## 15. Maintenance Principles

When maintaining this patch:

- Keep diffs small and focused on observed failures.
- Prefer retry, fallback, input constraints, and error classification over changing business data structures.
- Do not retry clearly unrecoverable errors such as authentication failures, exhausted quota, or user cancellation.
- Do not automatically retry after a model chunk has been emitted, to avoid duplicate side effects.
- Run `node --check` after every `extension.js` edit.
- Keep `README.md`, `README_EN.md`, and this document aligned when delivery behavior changes.
- Remind users to back up the original `extension.js` before replacement.

## 16. Rollback Flow

If the patch is incompatible with the current Kiro version:

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
Copy-Item -LiteralPath "$target.original-backup" -Destination $target -Force
node --check $target
```

Then reload or restart Kiro.

## 17. Current Non-Goals

The following are recorded but not fixed by this patch:

- Historical append action missing restore checkpoint:

```text
Cannot restore file ... original checkpoint missing for append action
```

- Bundle/source location metadata warning:

```text
No bundle location found for extension kiro.kiroAgent
```

- Patch migration after official Kiro bundle structure changes.
- Account quota exhaustion, authentication failures, or server-side policy restrictions.

These require broader state migration, upstream metadata fixes, or account/service-side handling. They should not be expanded into this local bundle patch without a clear reason.
