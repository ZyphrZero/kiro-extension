# Kiro Extension Local Patch Notes

This directory contains the Kiro agent extension bundle loaded by the local Kiro installation.

`extension.js` has been locally patched to work around write failures on the Windows filesystem, and to fix several categories of noisy logs and unstable behavior observed during Kiro spec/task execution. These changes are not official Kiro source modifications. Future Kiro updates may overwrite these patches.

## Deliverables

- `extension.js`
- `README.md`

Before replacing, users should back up the original `extension.js` from the Kiro installation directory, e.g. by copying it to:

- `extension.js.original-backup`

This way, if the patch is incompatible with the current Kiro version, the original file can be restored directly.

## Original Issues

### 1. Metadata writes fail with EPERM on Windows

Kiro task metadata updates write to:

```text
C:\Users\<USERNAME>\.kiro\tasks\<TASK_ID>\codex-provider.meta.json
```

The original write flow is:

1. Write a temporary file.
2. Immediately overwrite the target metadata file with `rename(temp, target)`.

On Windows, if Defender, an indexer, a file watcher, Kiro itself, or another process briefly holds the target file, `rename` may fail and throw:

- `EPERM`
- `EBUSY`
- `EACCES`

The original code has no retry and no fallback write path, so `taskUpdate` can fail repeatedly.

### 2. Missing explicit recovery guidance when write tool stream input is aborted

Write tools may produce errors like:

```text
Error(s) while editing, aborted
[fs_append] Stream error: type=StreamError message=Operation was aborted by user or system
```

When these errors occur, the model may not receive sufficiently explicit recovery guidance, so it may continue retrying with oversized write payloads, causing the same failure to repeat.

### 3. Write tool descriptions lack strong chunking constraints

The tool descriptions for `fs_write` and `fs_append` do not explicitly limit the size of a single write.

The model may pass very large content at once, increasing the risk of:

- Streamed tool parameters being aborted before fully arriving.
- Write parameter parsing failures.
- `fs_append` large-block appends failing more frequently.

### 4. Empty FileDecoration causes high-frequency extension host warnings

The logs repeatedly show:

```text
INVALID decoration from extension 'kiro.kiroAgent': Error: The decoration is empty
```

The root cause is that `AgentActivityFileDecorationProvider.provideFileDecoration()` returns an empty object or an object without valid `FileDecoration` display fields when there is no valid decoration content.

VS Code/Kiro's `FileDecoration` validation requires the decoration to contain at least one valid display field, such as:

- `badge`
- `tooltip`
- `color`

Otherwise it is classified as an empty decoration.

### 5. No automatic retry for model high-load and transient network errors

During peak model service traffic, you may see:

```text
The model you've selected is experiencing a high volume of traffic. Try changing the model and re-running your prompt.
```

The corresponding underlying errors in the logs are typically:

```text
Failed to stream response chunks I am experiencing high traffic, please try again shortly.
Encountered unexpectedly high load when processing the request, please try again.
```

Transient network disconnections may also produce:

```text
A network error occurred. Please check your connection and try again.
Failed to stream response chunks Client network socket disconnected before secure TLS connection was established
B14: aborted
```

The upper-level error translator may also unify some unknown errors as:

```text
An unexpected error occurred, please retry.
```

Spec task sub-agents may also display:

```text
Failed to invoke Spec Task Execution
```

This is typically not because `invoke_sub_agent` cannot find the sub-agent, but rather because the `spec-task-execution` sub-agent has encountered consecutive high-load/transient service errors internally, and after retries are exhausted, the parent tool wraps it as an invoke failure.

The original behavior marks the current execution as failed immediately. High load is typically a transient service capacity issue, and network errors may be brief jitter at the TLS/socket/fetch layer. Server messages containing `please retry` / `please try again` explicitly indicate that a retry should be attempted before surfacing the failure to the user.

### 6. `[Steering] ExistingFiles` file list bloat with duplicates

The logs show the same batch of files appearing multiple times:

```text
[Steering] ExistingFiles: ...
```

The root cause is that `getWorkspaceFiles()` collects file references from historical context, including:

- Previously injected file documents
- `read_file` tool use
- `read_files` tool use

The existing `removeDuplicateFiles()` only deduplicates `document` type entries and cannot deduplicate paths from tool use parameters, so the same path keeps entering `ExistingFiles` repeatedly.

This issue causes:

- Context bloat.
- Increased token pressure.
- Slower agent execution.
- Indirectly higher probability of streamed write aborts.

### 7. Stale diagnostics cause code problems formatting to fail entirely

The logs show:

```text
Caught error formatting code problems, skipping for now
```

The root cause is that the code problems formatter assumes `diagnostic.range.start.line` and `diagnostic.range.end.line` are always within the current document's line count.

When the agent rapidly rewrites a file, old diagnostics may have expired, with ranges pointing to lines that no longer exist in the current file. The original implementation throws an exception and skips the entire code problems formatting pass.

### 8. Historical append actions have missing restore checkpoint warnings

The logs also show:

```text
Cannot restore file ... original checkpoint missing for append action
```

This appears to be a state issue left over from previously aborted append actions or historical session migrations. Some append actions lack an original checkpoint, so restore can only log a warning.

This is not the primary cause of the current metadata `EPERM` or write failures. This patch does not modify restore/checkpoint semantics to avoid introducing larger diff recovery risks.

### 9. Bundle location warning may still persist

The extension host may still log:

```text
No bundle location found for extension kiro.kiroAgent
```

This is currently believed to be a metadata issue in the Kiro/VS Code fork regarding bundle/source location. The Kiro agent can still activate, execute, and write files normally. This patch does not modify this behavior.

## Applied Local Changes

### 1. Metadata file writes changed to Windows-safe writes

A Windows-aware write helper was added near the task metadata storage:

- Identifies transient Windows busy errors: `EPERM`, `EBUSY`, `EACCES`
- Up to 8 exponential-backoff retries for `rename(temp, target)`
- If rename is persistently blocked by busy errors, falls back to writing the target file directly
- Cleans up temporary files after failure or fallback

`TaskMetadataStorage.writeMetadataFile()` has been changed from directly calling `fs.promises.rename()` to using the safe write helper.

### 2. Enhanced write tool stream abort handling

`handleStreamError()` now additionally checks whether the error message contains `aborted`.

Even if the internal abort classifier does not recognize the error, as long as the error message indicates the stream input was aborted, the following tools will receive more explicit recovery guidance:

- `fs_write`
- `fs_append`
- `str_replace`

The hint returned to the model is:

```text
The streamed write input was aborted before complete. Retry with smaller chunks: keep fs_write/fs_append text to 40 lines or fewer, and use multiple str_replace calls for large edits.
```

This makes it easier for the model to automatically switch to smaller writes instead of continuing to retry with large blocks that fail.

### 3. Write tool descriptions updated with stronger chunking requirements

The descriptions for `fs_write` and `fs_append` have been updated to explicitly require:

- Regular single `fs_write` / `fs_append` text should be kept to 40 lines or fewer.
- For large files, use `fs_write` for the first chunk, then multiple `fs_append` calls for subsequent chunks.
- Large-scale edits should use multiple `str_replace` calls.

### 3.1 Early interception of streamed write tool parameters

`fs_write` and `fs_append` now check the `text` line count during the streamed parameter preview phase:

- If `text` exceeds 80 lines, the tool immediately returns a recoverable error.
- The error instructs the model to split the content into multiple `fs_write` / `fs_append` small chunks.
- This prevents the model from continuing to output oversized tool input until the upstream model stream is aborted after approximately 60 seconds, while the 80-line limit is not so narrow (like 25 lines) that it causes excessive tool calls.

### 3.2 Preventing repeated overwrites during tasks.md generation

The shared task generation prompts in `feature-requirements-first-workflow` / `feature-design-first-workflow` have been updated with the following constraints:

- When generating `tasks.md`, only one `fs_write` is allowed to create the header and first complete section.
- Subsequent sections must be appended with multiple `fs_append` calls.
- Each `fs_write` / `fs_append` `text` parameter should not exceed 40 lines, with an absolute maximum of 80 lines per call.
- Generating the complete `tasks.md` in a single `fs_write` or single `fs_append` is prohibited.
- After the first `fs_write` succeeds, subsequent `fs_write` calls to overwrite the entire `tasks.md` are prohibited within the same task generation process.
- Once `tasks.md` contains all required sections, the sub-agent must immediately call `subagent_response` and stop.

This change targets the actual error:

```text
The streamed write input was aborted before complete. Retry with smaller chunks...
```

Log evidence shows this error occurs inside the `tasks` preset of `invoke_sub_agent`, where the sub-agent generates `tasks.md` using `fs_write`, and the upstream `modelStream2` aborts before the streamed tool input completes.

Subsequent testing also found that if the limit is too tight at 25 lines, the `tasks` sub-agent may repeatedly overwrite `tasks.md` through multiple `fs_write` calls, causing the generation process to remain in a running state for an extended time. The current strategy is a 40-line recommended limit, 80-line hard cap, and explicit prohibition on repeatedly overwriting the entire file during `tasks.md` generation prompts.

### 4. Fixed FileDecoration return values

`AgentActivityFileDecorationProvider.provideFileDecoration()` now:

- Returns `undefined` when there is no decoration to display.
- Returns a valid decoration when there is agent activity:
  - `badge: "E"`
  - `tooltip: "Kiro agent activity: ..."`
  - `color: new ThemeColor("list.highlightForeground")`
  - Preserves `agentExecutionState`

This prevents empty decoration objects from triggering extension host validation warnings.

### 5. Automatic retry for model high-load and transient network errors

`It5._streamResponseChunks()` now performs limited automatic retries for safely recoverable stream errors:

- High-load exceptions: `Encountered unexpectedly high load when processing the request, please try again.`
- Transient network disconnection exceptions: `A network error occurred. Please check your connection and try again.`
- Network error codes: `B14`, `be10`.
- Common network error codes or messages: `ECONNRESET`, `ECONNABORTED`, `ECONNREFUSED`, `EHOSTUNREACH`, `ENETUNREACH`, `ENOTFOUND`, `EAI_AGAIN`, `ETIMEDOUT`, `ERR_NETWORK`, `fetch failed`, `failed to fetch`, `socket disconnected`.
- Transient server messages explicitly requesting retry: `please retry`, `please try again`, `An unexpected error occurred`, `temporarily unavailable`, `unexpectedly high load`.
- Default maximum of 5 retries, adjustable via constants in `extension.js`; infinite retry is not enabled by default to avoid tasks occupying execution state indefinitely during prolonged periods of high service load.
- Default backoff times are approximately 1.5s, 4s, 9s, 16s, 25s, with a small amount of random jitter.
- If the model has already streamed any chunks, automatic retry is skipped to avoid duplicate tool calls, duplicate file writes, or duplicate partial output submissions.
- If the retry limit is reached and still failing, the original error path is preserved and the error is surfaced to the user.
- Retry branch logs use `this.log.debug()`. This logger does not have a `warn()` method; using `warn()` would cause the retry logic itself to throw `this.log.warn is not a function`.

Users can search for and modify these constants in `extension.js`:

```js
KIRO_STREAM_RETRY_MAX_ATTEMPTS = 5;
KIRO_STREAM_RETRY_FOREVER = false;
KIRO_STREAM_RETRY_BACKOFF_MS = [1500, 4e3, 9e3, 16e3, 25e3];
```

Meanings:

- `KIRO_STREAM_RETRY_MAX_ATTEMPTS`: Maximum retry count, default `5`; only takes effect when `KIRO_STREAM_RETRY_FOREVER = false`; set to `0` to disable automatic retry.
- `KIRO_STREAM_RETRY_FOREVER`: Whether to retry indefinitely, default `false`; when set to `true`, `KIRO_STREAM_RETRY_MAX_ATTEMPTS` is ignored, but enabling this by default is not recommended.
- `KIRO_STREAM_RETRY_BACKOFF_MS`: Milliseconds to wait before each retry; when retrying indefinitely, the last wait time is reused once the array length is exceeded; if mistakenly set to an empty array, it falls back to 25 seconds.

This targets transient model capacity shortages and network jitter. It does not retry authentication failures, usage limits, overly long prompts, user cancellations, parameter validation failures, or other non-transient errors. The error object's `cause` chain is checked up to 10 levels deep to avoid getting stuck due to exception object cycles.

### 6. `getWorkspaceFiles()` deduplicates by path before returning

`getWorkspaceFiles()` now deduplicates by `path` before returning:

- Keeps the first occurrence of each path.
- Discards subsequent duplicate paths.
- Prevents historical `read_file` / `read_files` calls from continuously inflating `[Steering] ExistingFiles`.

### 7. Code problems formatter skips out-of-bounds diagnostics

The code problems formatter now skips diagnostics that:

- Have no `range`
- Have `start.line` less than 0
- Have `end.line` less than 0
- Have `start.line` exceeding the current document's line count
- Have `end.line` exceeding the current document's line count

This prevents a single stale diagnostic from causing the entire code problems formatting pass to fail.

## Verification Performed

The main bundle has passed syntax checking:

```powershell
node --check "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
```

Post-patch confirmation:

- The metadata safe write helper is still present.
- No residual `codex-provider.meta.json*.tmp` files in the affected task metadata directory.
- No new `EPERM`, `EACCES`, or `EBUSY` metadata write failures observed in recent logs.
- Multiple successful file write records confirmed in Kiro logs.

## Activation Requirements

Kiro must reload the extension host before these changes take effect.

Use either of the following methods:

- Execute `Developer: Reload Window` in Kiro
- Fully restart Kiro

Before reloading, the existing extension host still runs the old `extension.js` in memory, so logs may continue to show old behavior. This does not mean the file patch was not applied.

After reloading, it is recommended to check:

- `exthost.log` should no longer have new entries for:

```text
INVALID decoration from extension 'kiro.kiroAgent': Error: The decoration is empty
```

- `[Steering] ExistingFiles` in `Kiro Logs.log` should no longer list duplicate paths.
- Metadata writes should no longer report `EPERM`, `EACCES`, or `EBUSY`.

## Ongoing Maintenance Notes

- These are local bundle file patches, not official Kiro source modifications.
- Kiro application updates may overwrite `extension.js`.
- If a Kiro update overwrites the patch, refer to this README for re-comparison; if incompatible, restore using the user's backed-up original `extension.js`.
- Run `node --check` after every bundle edit.
