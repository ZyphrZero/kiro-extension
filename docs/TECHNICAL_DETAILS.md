# Kiro Agent Extension 本地补丁技术细节

本文档面向维护者、排障人员和需要二次修改 `extension.js` 的用户。根目录 `README.md` 只保留安装和概要说明，本文档记录补丁的设计动机、关键实现、验证方式和已知边界。

## 1. 文档范围

本仓库维护的是 Kiro 已打包后的 agent 扩展 bundle：

```text
extension.js
```

它不是 Kiro 官方 TypeScript 源码，也不是可重新构建的源码工程。当前补丁直接作用于 bundle 中已编译、已打包的 JavaScript 代码，因此维护时需要以稳定搜索锚点为主，而不是依赖上游源码文件路径。

补丁主要解决 Windows 环境下 Kiro spec/task 执行时的几类实际问题：

- `taskUpdate` 写 metadata 时遇到 `EPERM`、`EBUSY`、`EACCES`。
- `fs_write` / `fs_append` 大块流式参数未传完即被 abort。
- `tasks.md` 生成阶段重复全量覆盖文件，导致子任务长时间 running。
- 模型高负载、临时限流、网络瞬断时任务直接失败。
- 模型用 `Start-Sleep -Seconds ...; Write-Output "ok"` 做长时间 shell 空等，而不是交给扩展内部 retry 或报告重试耗尽。
- extension host 高频记录空 `FileDecoration` 警告。
- `[Steering] ExistingFiles` 上下文文件列表重复膨胀。
- stale diagnostics 越界导致 code problems 整体格式化失败。

## 2. 交付和目录约定

对外交付只应包含：

```text
extension.js
README.md
```

如果需要同时交付技术说明，可以附带 `docs/TECHNICAL_DETAILS.md`，但不要把本机生成的备份文件一起分发。

用户替换前必须自行备份安装目录中的原版 bundle，例如：

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
Copy-Item -LiteralPath $target -Destination "$target.original-backup"
```

原因是 Kiro 更新后官方 bundle 可能变化，补丁 bundle 不一定适配所有版本。保留原版文件可以快速回滚。

## 3. 运行环境和安装位置

典型 Kiro 安装目录：

```text
D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js
```

典型任务 metadata 目录：

```text
C:\Users\<USERNAME>\.kiro\tasks\<WORKSPACE_HASH>\<PROVIDER>.meta.json
```

例如：

```text
C:\Users\<USERNAME>\.kiro\tasks\<TASK_DIR>\codex-provider.meta.json
```

补丁的主要目标平台是 Windows。部分逻辑只在 `process.platform === "win32"` 时启用，避免改变非 Windows 平台上的文件写入语义。

## 4. 补丁总览

| 模块 | 搜索锚点 | 主要行为 |
| --- | --- | --- |
| Task metadata 写入 | `writeMetadataFile`、`EPERM`、`EBUSY`、`EACCES` | 对 Windows transient busy 错误做 retry，必要时 fallback 直接写目标文件 |
| 流式写入错误处理 | `handleStreamError`、`The streamed write input was aborted before complete` | 把 aborted 写入错误转换为明确的小块重试建议 |
| 写入工具描述和预检 | `fs_write text is too large`、`fs_append text is too large` | 40 行常规建议，80 行硬上限 |
| `tasks.md` 生成提示 | `The model MUST write tasks.md incrementally` | 禁止一次性生成或反复全量覆盖 `tasks.md` |
| 模型流重试 | `KIRO_STREAM_RETRY_MAX_ATTEMPTS`、`getRetryableStreamErrorKind` | 高负载、限流、网络瞬断在首 chunk 前有限重试 |
| shell 空等拦截 | `KIRO_getBlockedNoopWaitReason`、`Blocked no-op shell wait command` | 拒绝长时间 `Start-Sleep` 加 `Write-Output "ok"` / `"retry"` 的空等退避 |
| FileDecoration | `AgentActivityFileDecorationProvider`、`provideFileDecoration` | 无装饰时返回 `undefined`，有活动时返回合法 decoration |
| Workspace file 去重 | `getWorkspaceFiles()`、`read_file`、`read_files` | 返回前按 path 去重 |
| Diagnostics 格式化 | `Caught error formatting code problems` | 跳过 stale/out-of-range diagnostic |

## 5. Windows metadata 写入补丁

### 5.1 原始问题

Kiro task 状态更新会反复写 metadata 文件。原始流程通常是：

1. 写入临时文件。
2. 使用 `rename(temp, target)` 替换目标 metadata 文件。

在 Windows 上，`rename` 对文件占用更敏感。如果 Defender、索引器、文件 watcher、Kiro 自身或其他进程短暂持有目标文件句柄，`rename` 可能抛出：

```text
EPERM
EBUSY
EACCES
```

这些错误往往是瞬时状态，不代表文件永久不可写。原实现没有 retry 和 fallback，导致 `taskUpdate` 可能直接失败。

### 5.2 当前实现

可在 bundle 中搜索：

```text
task-metadata-storage
writeMetadataFile
EPERM
EBUSY
EACCES
```

关键补丁逻辑包含四个 helper：

| Helper | 作用 |
| --- | --- |
| `P18(error)` | 判断是否为 Windows transient busy 错误 |
| `B11(ms)` | sleep helper |
| `G11(fn)` | 对 transient busy 错误做最多 8 次指数退避 retry |
| `K11(target, temp, content)` | 先写 temp，再 retry rename；rename 持续 busy 时 fallback 直接写 target |

当前策略：

1. `writeMetadataFile()` 仍先创建 task metadata 目录。
2. 生成随机后缀临时文件，避免多个写入共享固定 temp path。
3. 写入 temp 文件。
4. 对 `rename(temp, target)` 做最多 8 次 retry。
5. 如果 `rename` 最终仍因 Windows busy 错误失败，则 fallback 到直接写目标文件。
6. fallback 后尝试删除 temp 文件。
7. 如果错误不是 `EPERM`、`EBUSY`、`EACCES`，不吞掉，继续抛出。

### 5.3 设计取舍

这个补丁没有引入全局锁，也没有改变 metadata JSON 结构。原因是当前问题是 Windows 文件系统的短暂占用，而不是 metadata 业务模型错误。

fallback 直接写目标文件不是首选路径，只在 `rename` 被持续阻塞时使用。这样优先保留原始 atomic rename 的语义，同时给 Windows 上的 transient busy 一个恢复路径。

### 5.4 验证方式

安装后观察 Kiro 日志，重点确认不再出现连续的：

```text
codex-provider.meta.json
EPERM
EBUSY
EACCES
taskUpdate
```

语法检查：

```powershell
node --check "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
```

## 6. 流式写入 abort 处理

### 6.1 原始问题

`fs_write` / `fs_append` 的参数是模型流式输出的一部分。如果模型一次性输出很长的 `text` 参数，可能出现：

```text
The streamed write input was aborted before complete.
Operation was aborted by user or system
Error(s) while editing, aborted
```

原始错误对模型来说不够可操作，模型可能继续用同样的大块内容重试，导致重复失败。

### 6.2 当前实现

可搜索：

```text
handleStreamError
The streamed write input was aborted before complete
```

当 `handleStreamError()` 看到错误 message 包含 `aborted`，并且当前工具是：

```text
fs_write
fs_append
str_replace
```

会返回更明确的恢复提示：

```text
The streamed write input was aborted before complete. Retry with smaller chunks: keep fs_write/fs_append text to 40 lines or fewer, and use multiple str_replace calls for large edits.
```

这个提示进入工具错误消息，让模型在下一步更容易切换到小块写入或多次 `str_replace`。

### 6.3 为什么不是 25 行

曾经考虑过更小的 25 行限制，但实际测试发现过窄限制会导致工具调用次数大幅增加。尤其在生成 `tasks.md` 时，子代理可能多次 `fs_write` 覆盖同一文件，造成任务长时间处于 running。

当前采用：

- 40 行：常规建议值。
- 80 行：硬性预检上限。

这个组合减少 abort 风险，同时避免工具调用过度碎片化。

## 7. `fs_write` / `fs_append` 分块策略

### 7.1 工具描述补丁

可搜索：

```text
RELIABILITY LIMIT
Keep text to
Very large streamed inputs are rejected before execution
```

`fs_write` 和 `fs_append` 的工具描述被增强为：

- 常规单次写入控制在 40 行以内。
- 大文件先用一次 `fs_write` 写第一块。
- 后续内容用多次 `fs_append` 追加。
- 大范围编辑优先拆成多次 `str_replace`。

### 7.2 streamed parameter preview 预检

可搜索：

```text
fs_write text is too large for reliable streaming
fs_append text is too large for reliable streaming
```

工具在 streamed parameter preview 阶段检查 `text` 行数：

- 如果 `text` 超过 80 行，直接抛出 recoverable error。
- 这发生在真正执行文件写入之前。
- 目的不是禁止写大文件，而是要求模型拆成多次小块写入。

### 7.3 行数规则

| 场景 | 推荐 |
| --- | --- |
| 普通 `fs_write` / `fs_append` | 不超过 40 行 |
| 绝对上限 | 不超过 80 行 |
| 新建大文件 | 首块 `fs_write`，后续 `fs_append` |
| 修改大文件 | 多次 `str_replace` |
| 生成 `tasks.md` | 一次 `fs_write` 创建开头，后续 `fs_append` |

## 8. `tasks.md` 生成约束

### 8.1 原始问题

spec task 子代理生成 `tasks.md` 时，如果一次性写入完整文件，容易触发流式参数 abort。若限制过窄，又可能出现多次 `fs_write` 重复覆盖全文件，导致子任务一直 running。

### 8.2 当前实现

可搜索：

```text
The model MUST write tasks.md incrementally
After the first successful fs_write for tasks.md
```

补丁在 task generation prompt 中增加约束：

- 生成 `tasks.md` 时，只允许一次 `fs_write` 创建标题和第一个完整 section。
- 后续 section 必须使用 `fs_append`。
- 单次 `fs_write` / `fs_append` 常规不超过 40 行，硬上限 80 行。
- 禁止用单次 `fs_write` 或单次 `fs_append` 生成完整 `tasks.md`。
- 首次 `fs_write` 成功后，禁止再次对 `tasks.md` 使用 `fs_write` 全量覆盖。
- 文件包含所有必要 section 后，子代理应调用 `subagent_response` 并停止。

### 8.3 预期效果

这个补丁降低两类风险：

- 大块 tool input 没传完就 abort。
- 子代理通过重复覆盖同一文件进入长时间 running。

## 9. 模型高负载、限流和网络错误重试

### 9.1 原始问题

Kiro 模型流可能在首个 chunk 前失败，常见表现包括：

```text
The model you've selected is experiencing a high volume of traffic. Try changing the model and re-running your prompt.
An unexpected error occurred, please retry.
A network error occurred. Please check your connection and try again.
Too many requests, please wait before trying again.
Failed to invoke Spec Task Execution
```

其中 `Failed to invoke Spec Task Execution` 往往不是子代理不存在，而是子代理内部连续遇到模型高负载、临时限流或网络瞬断，最终被父级工具包装成 invoke 失败。

### 9.2 当前实现

可搜索：

```text
KIRO_STREAM_RETRY_MAX_ATTEMPTS
KIRO_STREAM_RETRY_FOREVER
KIRO_STREAM_RETRY_BACKOFF_MS
_streamResponseChunks
getRetryableStreamErrorKind
```

核心常量：

```js
KIRO_STREAM_RETRY_MAX_ATTEMPTS = 5;
KIRO_STREAM_RETRY_FOREVER = false;
KIRO_STREAM_RETRY_BACKOFF_MS = [1500, 4e3, 9e3, 16e3, 25e3];
```

`_streamResponseChunks()` 包装 `_streamResponseChunksOnce()`：

1. 如果调用成功产出 chunk，则正常返回。
2. 如果首个 chunk 前抛错，调用 `getRetryableStreamErrorKind()` 判断是否可重试。
3. 如果错误不可重试，直接抛出。
4. 如果已经产出过任何 chunk，不再自动重试。
5. 如果可重试且未超过次数，按 backoff 等待后重试。
6. 达到次数上限后保留原错误路径。

### 9.3 会重试的错误

当前会进入有限重试的类别：

| 类别 | 例子 |
| --- | --- |
| 模型高负载 | `Encountered unexpectedly high load when processing the request, please try again.` |
| Kiro 内部 throttling | `retryErrorType === "THROTTLING"` |
| 临时限流 | `Too many requests, please wait before trying again.` |
| 网络错误类 | `B14`、`be10` |
| 网络错误码 | `ECONNRESET`、`ECONNABORTED`、`ECONNREFUSED`、`EHOSTUNREACH`、`ENETUNREACH`、`ENOTFOUND`、`EAI_AGAIN`、`ETIMEDOUT`、`ERR_NETWORK` |
| 网络错误 message | `network error`、`failed to fetch`、`fetch failed`、`socket disconnected` |
| 服务端明确建议重试 | `please retry`、`please try again`、`temporarily unavailable`、`An unexpected error occurred` |

为了兼容日志或复制时的空格异常，也匹配：

```text
please wait beforetrying again
```

### 9.4 不会重试的错误

以下错误不会自动重试：

- 鉴权失败。
- Kiro access 不可用。
- hourly/daily/weekly/monthly usage limit。
- overage limit。
- prompt 或 context 超长。
- 参数校验失败。
- 用户取消。
- 已经输出过任何模型 chunk 后的失败。

原因是这些错误通常不是短暂服务抖动。盲目重试会浪费请求次数，甚至可能造成重复工具调用。

### 9.5 为什么只在首个 chunk 前重试

模型一旦产出 chunk，后续可能已经触发工具调用、文件写入或部分用户可见输出。此时自动重试会带来风险：

- 重复写文件。
- 重复执行工具。
- 生成重复或互相冲突的回复。
- 难以判断前一次输出是否已经产生副作用。

因此补丁只在首个 stream chunk 之前进行自动 retry。

### 9.6 无限重试开关

`KIRO_STREAM_RETRY_FOREVER` 保持默认 `false`。

不建议默认设为 `true`，原因：

- 服务长时间高负载时，任务会无限占用 running 状态。
- 免费账号或临时限流可能持续较久，无限重试会掩盖真实状态。
- 用户可能误以为任务仍在有效推进。

如果用户明确希望长时间等待，可以手动改为：

```js
KIRO_STREAM_RETRY_FOREVER = true;
```

同时应理解它会忽略 `KIRO_STREAM_RETRY_MAX_ATTEMPTS`。

## 10. shell 空等命令拦截

### 10.1 原始问题

当模型高负载、限流或子代理失败已经冒泡到 agent 时，模型可能自己调用 `execute_pwsh` 做长时间等待，例如：

```powershell
Start-Sleep -Seconds 360; Write-Output "ok"
```

这不是扩展写死的命令，而是模型把 shell 当成 retry/backoff 计时器使用。实际效果是任务长时间停留在 running，但没有执行任何有价值的检查或修改。

### 10.2 当前实现

可搜索：

```text
KIRO_getBlockedNoopWaitReason
Blocked no-op shell wait command
Do not use Start-Sleep plus Write-Output
```

补丁在两套 shell 工具执行路径中都加入了执行前检查。匹配以下模式时会直接拒绝执行：

```powershell
Start-Sleep -Seconds <N>; Write-Output "ok"
Start-Sleep -Seconds <N>; Write-Output "retry"
Start-Sleep <N>; echo ok
powershell -Command "Start-Sleep -Seconds <N>; Write-Output 'ok'"
```

当前只拦截 `N >= 60` 的 no-op wait。被拦截后，工具返回可恢复错误，提示模型不要用 shell 空等处理高负载、限流或网络错误，而是依赖扩展内部 stream retry；如果重试耗尽，就直接向用户报告。

### 10.3 不会拦截的情况

以下命令不会被这个 guard 拦截：

- 短等待，例如 `Start-Sleep -Seconds 2`。
- 等待后执行真实检查，例如 `Start-Sleep -Seconds 2; Get-ChildItem ...`。
- 开发服务器、watcher 等真正的长运行命令，它们仍由已有 long-running command 规则处理。

这个限制只针对“长时间等待后输出 ok/retry/done”的空等退避，避免误伤正常诊断。

## 11. FileDecoration 空对象修复

### 11.1 原始问题

extension host 可能反复记录：

```text
INVALID decoration from extension 'kiro.kiroAgent': Error: The decoration is empty
```

原因是 `provideFileDecoration()` 在没有有效 decoration 时返回空对象，或返回缺少展示字段的对象。

### 11.2 当前实现

可搜索：

```text
AgentActivityFileDecorationProvider
provideFileDecoration
Kiro agent activity
```

当前行为：

- 如果 decoration 不可见，返回 `undefined`。
- 如果文件没有 agent activity，返回 `undefined`。
- 如果有 activity，返回合法对象：

```js
{
  badge: "E",
  tooltip: "Kiro agent activity: ...",
  color: new ThemeColor("list.highlightForeground"),
  agentExecutionState: state
}
```

`badge`、`tooltip`、`color` 至少提供了 VS Code/Kiro `FileDecoration` 校验所需的显示字段。

## 12. `getWorkspaceFiles()` 去重

### 12.1 原始问题

Kiro 上下文会从多处收集文件引用：

- 显式传入的 file document。
- `read_file` 工具调用。
- `read_files` 工具调用。
- 历史上下文。

原有 `removeDuplicateFiles()` 只处理 document 类型，无法去重 tool use 参数里的路径，因此日志中可能出现重复膨胀：

```text
[Steering] ExistingFiles: ...
```

### 12.2 当前实现

可搜索：

```text
getWorkspaceFiles()
read_file
read_files
InvalidFilePathInReadFile
```

当前 `getWorkspaceFiles()` 先按原逻辑收集文件，再用 `Set` 按 `path` 过滤：

- 保留第一次出现的路径。
- 丢弃后续重复路径。
- 不改变路径字符串本身。
- 不尝试做大小写归一、realpath 解析或 symlink 解析。

### 12.3 设计边界

当前去重是保守的字符串去重。它不会把下面这些视作同一个文件：

```text
C:\Project\File.ts
c:\project\file.ts
..\project\file.ts
```

这样可以避免在 bundle 补丁中引入路径规范化副作用。

## 13. stale diagnostics 跳过

### 13.1 原始问题

快速编辑文件后，旧 diagnostics 可能指向已经不存在的行。原始 formatter 直接访问 `diagnostic.range.start.line` 对应的文本行，越界时可能抛出，最终整段 code problems 格式化失败：

```text
Caught error formatting code problems, skipping for now
```

### 13.2 当前实现

可搜索：

```text
Caught error formatting code problems
diagnostic.range.start.line
diagnostic.range.end.line
```

当前逻辑在处理每条 diagnostic 前检查：

- `range` 必须存在。
- `start.line` 和 `end.line` 不能为负数。
- `start.line` 和 `end.line` 必须小于当前文档行数。

不满足条件的 diagnostic 会被跳过，不再导致整个 code problems block 失败。

## 14. 安装验证流程

### 14.1 替换前

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
Copy-Item -LiteralPath $target -Destination "$target.original-backup"
node --check ".\extension.js"
```

### 14.2 替换

```powershell
Copy-Item -LiteralPath ".\extension.js" -Destination $target -Force
```

如果失败：

1. 完全退出 Kiro。
2. 确认没有残留 Kiro 进程。
3. 重新执行复制。
4. 仍失败时使用管理员 PowerShell。

### 14.3 替换后

```powershell
node --check $target
Select-String -Path $target -Pattern "KIRO_STREAM_RETRY_MAX_ATTEMPTS","KIRO_STREAM_RETRY_FOREVER","KIRO_STREAM_RETRY_BACKOFF_MS"
```

然后执行：

- `Developer: Reload Window`
- 或完全重启 Kiro。

### 14.4 功能观察点

启动后观察 Kiro 日志：

```text
%APPDATA%\Kiro\logs
```

重点确认：

- 不再持续出现 empty decoration warning。
- metadata 写入不再因 `EPERM`、`EBUSY`、`EACCES` 直接失败。
- 高负载、限流、网络瞬断先出现 retry debug log，而不是立刻中断。
- `Failed to invoke Spec Task Execution` 的频率下降。
- `[Steering] ExistingFiles` 不再出现同一路径大量重复。

## 15. 常见问题排障

### 15.1 仍然看到 `Too many requests`

如果达到 5 次重试后服务仍限流，最终仍会显示错误。这是预期行为。默认补丁不是无限重试。

可检查：

```powershell
Select-String -Path $target -Pattern "retryErrorType === `"THROTTLING`"","Too many requests"
```

如果需要更长等待，可以调整：

```js
KIRO_STREAM_RETRY_MAX_ATTEMPTS = 10;
```

不建议默认打开：

```js
KIRO_STREAM_RETRY_FOREVER = true;
```

### 15.2 仍然出现用量上限错误

例如：

```text
You've reached your hourly usage limit.
You've reached your daily usage limit.
You've reached your monthly usage limit.
Overage limit reached
```

这些不是临时网络或服务抖动，不会自动重试。需要等待额度恢复或更换账号/计划。

### 15.3 `Failed to invoke Spec Task Execution` 仍出现

该错误只是父级工具包装后的结果，需继续看前面的真实错误：

- 高负载。
- 临时限流。
- 网络中断。
- 子代理生成 `tasks.md` 写入失败。
- 权限或鉴权错误。

如果前置错误是高负载、限流、网络瞬断，补丁会先有限重试。如果重试耗尽仍失败，父级仍可能显示 invoke failure。

### 15.4 文件写入仍被 abort

检查模型是否仍在尝试一次性写入超大 `text`：

- 单次 `fs_write` / `fs_append` 是否超过 80 行。
- 是否反复对同一 `tasks.md` 使用 `fs_write`。
- 是否应该改为多次 `str_replace`。

### 15.5 仍然看到长时间 `Start-Sleep`

如果命令是下面这种 no-op wait，应该被拒绝：

```powershell
Start-Sleep -Seconds 360; Write-Output "ok"
```

检查安装目录中的 bundle 是否包含：

```powershell
Select-String -Path $target -Pattern "KIRO_getBlockedNoopWaitReason","Blocked no-op shell wait command"
```

如果没有，说明安装目录还没有替换到最新补丁，或 Kiro 更新覆盖了补丁。

如果命令是短等待后做真实检查，例如：

```powershell
Start-Sleep -Seconds 2; Get-ChildItem ...
```

这是允许的，因为它不是空等退避。

### 15.6 Kiro 更新后补丁失效

Kiro 更新可能覆盖安装目录中的 `extension.js`。更新后需要：

1. 确认安装目录中的 `extension.js` 是否仍含 `KIRO_STREAM_RETRY_MAX_ATTEMPTS`。
2. 如果不存在，说明补丁被覆盖。
3. 重新备份新的官方 bundle。
4. 再替换补丁 bundle。

## 16. 维护原则

维护这个补丁时建议遵守以下原则：

- 保持差异小，只修复已经观察到的问题。
- 优先增加 retry、fallback、输入约束和错误分类，不改变业务数据结构。
- 不重试明显不可恢复的错误，例如鉴权失败、额度耗尽、用户取消。
- 不在已产生模型 chunk 后自动重试，避免重复副作用。
- 每次修改 `extension.js` 后必须运行 `node --check`。
- 修改交付行为时同步更新 `README.md` 和本文档。
- 分发时提醒用户自行备份原版 `extension.js`。

## 17. 回滚流程

如果补丁不适配当前 Kiro 版本：

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
Copy-Item -LiteralPath "$target.original-backup" -Destination $target -Force
node --check $target
```

然后 reload 或重启 Kiro。

## 18. 当前不会处理的问题

以下问题当前只记录，不在本补丁范围内修复：

- 历史 append action 缺少 restore checkpoint：

```text
Cannot restore file ... original checkpoint missing for append action
```

- bundle/source location 元数据警告：

```text
No bundle location found for extension kiro.kiroAgent
```

- 官方 Kiro 版本升级后 bundle 结构变化导致的补丁迁移。
- 账号额度耗尽、鉴权失败或服务端策略限制。

这些问题需要更大范围的状态迁移、上游元数据修复或账号/服务侧处理，不适合在本地 bundle 补丁中直接扩大改动面。
