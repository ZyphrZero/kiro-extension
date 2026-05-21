# Kiro 扩展本地补丁说明

这个目录包含本机 Kiro 安装包加载的 Kiro agent 扩展 bundle。

`extension.js` 已被打了本地补丁，用来规避 Windows 文件系统上的写入失败，并修复 Kiro spec/task 执行时观察到的几类噪声日志和不稳定行为。这些改动不是 Kiro 官方源码改动。后续 Kiro 更新可能会覆盖这些补丁。

## 交付文件

- `extension.js`
- `README.md`

用户替换前应先自行备份 Kiro 安装目录里的原版 `extension.js`，例如复制为：

- `extension.js.original-backup`

这样如果补丁不适配当前 Kiro 版本，可以直接恢复原版文件。

## 原始问题

### 1. Windows 上 metadata 写入会因为 EPERM 失败

Kiro task metadata 更新时会写入：

```text
C:\Users\cassianvale\.kiro\tasks\9149b1d5febb49c1\codex-provider.meta.json
```

原实现的写入流程是：

1. 写一个临时文件。
2. 立即用 `rename(temp, target)` 覆盖目标 metadata 文件。

在 Windows 上，如果 Defender、索引器、文件 watcher、Kiro 自身或其他进程短暂占用目标文件，`rename` 可能失败，并抛出：

- `EPERM`
- `EBUSY`
- `EACCES`

原代码没有重试，也没有 fallback 写入路径，所以 `taskUpdate` 可能反复失败。

### 2. 写入工具流式输入被中止时缺少明确恢复建议

写入工具可能出现类似错误：

```text
Error(s) while editing, aborted
[fs_append] Stream error: type=StreamError message=Operation was aborted by user or system
```

这类错误发生时，模型原本不一定能收到足够明确的恢复建议，因此可能继续用过大的写入 payload 重试，导致失败重复出现。

### 3. 写入工具描述没有足够强的分块约束

`fs_write` 和 `fs_append` 的工具说明没有明确限制单次写入规模。

模型可能一次性传入很长的内容，增加这些风险：

- 流式 tool 参数还没完整到达就被 abort。
- 写入参数解析失败。
- `fs_append` 大块追加时更容易失败。

### 4. 空 FileDecoration 导致 extension host 高频警告

日志中反复出现：

```text
INVALID decoration from extension 'kiro.kiroAgent': Error: The decoration is empty
```

根因是 `AgentActivityFileDecorationProvider.provideFileDecoration()` 在没有有效装饰内容时返回了空对象，或者返回了不包含有效 `FileDecoration` 展示字段的对象。

VS Code/Kiro 的 `FileDecoration` 校验要求 decoration 至少包含有效显示信息，例如：

- `badge`
- `tooltip`
- `color`

否则会被判定为 empty decoration。

### 5. 模型高负载和瞬时网络错误没有自动重试

模型服务高峰期可能出现：

```text
The model you've selected is experiencing a high volume of traffic. Try changing the model and re-running your prompt.
```

日志中对应的底层错误通常是：

```text
Failed to stream response chunks I am experiencing high traffic, please try again shortly.
Encountered unexpectedly high load when processing the request, please try again.
```

网络链路瞬断时也可能出现：

```text
A network error occurred. Please check your connection and try again.
Failed to stream response chunks Client network socket disconnected before secure TLS connection was established
B14: aborted
```

上层错误翻译器还可能把部分未知错误统一显示为：

```text
An unexpected error occurred, please retry.
```

原行为是直接把当前执行标记为失败。高负载通常是瞬时服务容量问题，网络错误也可能只是 TLS/socket/fetch 层的短暂抖动，带有 `please retry` / `please try again` 的服务端文案也明确表示应先重试，再把失败暴露给用户。

### 6. `[Steering] ExistingFiles` 文件列表重复膨胀

日志中可以看到同一批文件重复出现多次：

```text
[Steering] ExistingFiles: ...
```

根因是 `getWorkspaceFiles()` 会从历史上下文中收集文件引用，包括：

- 之前注入的 file document
- `read_file` tool use
- `read_files` tool use

已有的 `removeDuplicateFiles()` 只去重 `document` 类型，无法去重 tool use 参数里的路径，所以同一个路径会不断重复进入 `ExistingFiles`。

这个问题会导致：

- 上下文膨胀。
- token 压力增加。
- agent 执行变慢。
- 间接提高流式写入中止概率。

### 7. stale diagnostics 会导致 code problems 格式化整体失败

日志中出现：

```text
Caught error formatting code problems, skipping for now
```

根因是 code problems formatter 假设 `diagnostic.range.start.line` 和 `diagnostic.range.end.line` 一定还在当前文档行数范围内。

当 agent 快速重写文件后，旧 diagnostics 可能已经过期，range 指向了当前文件不存在的行。原实现会因此抛异常，并跳过整段 code problems 格式化。

### 8. 历史 append action 存在 restore checkpoint 缺失警告

日志中还出现：

```text
Cannot restore file ... original checkpoint missing for append action
```

这看起来是之前 aborted append 或历史 session 迁移留下的状态问题。某些 append action 没有 original checkpoint，所以 restore 时只能记录警告。

这不是当前 metadata `EPERM` 或写入失败的主因。本补丁没有修改 restore/checkpoint 语义，避免引入更大的 diff 恢复风险。

### 9. bundle location 警告仍可能存在

extension host 可能仍然记录：

```text
No bundle location found for extension kiro.kiroAgent
```

目前判断这是 Kiro/VS Code fork 的 bundle/source location 元数据问题。Kiro agent 实际仍能激活、执行和写文件。本补丁没有修改这个行为。

## 已应用的本地改动

### 1. metadata 文件写入改为 Windows 安全写入

在 task metadata storage 附近新增了 Windows-aware 写入 helper：

- 识别瞬时 Windows busy 错误：`EPERM`、`EBUSY`、`EACCES`
- 对 `rename(temp, target)` 做最多 8 次指数退避重试
- 如果 rename 持续被 busy 错误阻塞，则 fallback 为直接写目标文件
- 失败或 fallback 后清理临时文件

`TaskMetadataStorage.writeMetadataFile()` 已从直接调用 `fs.promises.rename()` 改为走安全写入 helper。

### 2. 写入工具 stream abort 处理增强

`handleStreamError()` 现在会额外检查错误 message 中是否包含 `aborted`。

即使内部 abort classifier 没识别出该错误，只要错误信息表示流式输入被中止，以下工具都会收到更明确的恢复建议：

- `fs_write`
- `fs_append`
- `str_replace`

返回给模型的提示为：

```text
The streamed write input was aborted before complete. Retry with smaller chunks: keep fs_write/fs_append text to 40 lines or fewer, and use multiple str_replace calls for large edits.
```

这样模型更容易自动改用小块写入，而不是继续重复大块失败。

### 3. 写入工具说明加入更稳的分块要求

`fs_write` 和 `fs_append` 的描述被更新，明确要求：

- 常规单次 `fs_write` / `fs_append` 的 text 建议保持在 40 行以内。
- 大文件先用 `fs_write` 写第一块，再用多个 `fs_append` 追加后续块。
- 大规模编辑使用多个 `str_replace` 调用。

### 3.1 写入工具流式参数提前拦截

`fs_write` 和 `fs_append` 现在会在流式参数预览阶段检查 `text` 行数：

- 如果 `text` 超过 80 行，工具会立即返回可恢复错误。
- 错误会要求模型把内容拆成多个 `fs_write` / `fs_append` 小块。
- 这样可以避免模型继续输出超大 tool input，直到上游模型流在约 60 秒后被 abort，同时不会因为 25 行过窄导致工具调用过于频繁。

### 3.2 tasks.md 生成阶段避免反复覆盖

`feature-requirements-first-workflow` / `feature-design-first-workflow` 共用的任务生成提示词中，已新增约束：

- 生成 `tasks.md` 时必须只用一次 `fs_write` 创建标题和第一个完整章节。
- 后续章节必须用多个 `fs_append` 追加。
- 常规每个 `fs_write` / `fs_append` 的 `text` 参数建议不超过 40 行，单次绝对不超过 80 行。
- 禁止用一次 `fs_write` 或一次 `fs_append` 生成完整 `tasks.md`。
- 第一次 `fs_write` 成功后，同一次任务生成过程中禁止再次对 `tasks.md` 调用 `fs_write` 覆盖全文件。
- `tasks.md` 包含所有必需章节后，子代理必须立即调用 `subagent_response` 并停止。

这次改动针对的实际错误是：

```text
The streamed write input was aborted before complete. Retry with smaller chunks...
```

日志证据显示该错误发生在 `invoke_sub_agent` 的 `tasks` preset 内部，子代理生成 `tasks.md` 时使用了 `fs_write`，上游 `modelStream2` 在流式 tool input 尚未完成时 abort。

后续测试还发现，如果把限制收得过窄到 25 行，`tasks` 子代理可能通过多次 `fs_write` 反复覆盖 `tasks.md`，导致生成过程长时间处于 running。当前策略改为 40 行常规建议、80 行硬保险，并在 `tasks.md` 生成提示中明确禁止反复覆盖全文件。

### 4. 修复 FileDecoration 返回值

`AgentActivityFileDecorationProvider.provideFileDecoration()` 现在：

- 没有 decoration 需要显示时返回 `undefined`。
- 有 agent activity 时返回合法 decoration：
  - `badge: "E"`
  - `tooltip: "Kiro agent activity: ..."`
  - `color: new ThemeColor("list.highlightForeground")`
  - 保留 `agentExecutionState`

这避免了空 decoration 对象触发 extension-host 校验警告。

### 5. 模型高负载和瞬时网络错误自动重试

`It5._streamResponseChunks()` 现在会对可安全恢复的 stream 错误做有限自动重试：

- 高负载异常：`Encountered unexpectedly high load when processing the request, please try again.`
- 网络瞬断异常：`A network error occurred. Please check your connection and try again.`
- 网络错误类：`B14`、`be10`。
- 常见网络错误码或 message：`ECONNRESET`、`ECONNABORTED`、`ECONNREFUSED`、`EHOSTUNREACH`、`ENETUNREACH`、`ENOTFOUND`、`EAI_AGAIN`、`ETIMEDOUT`、`ERR_NETWORK`、`fetch failed`、`failed to fetch`、`socket disconnected`。
- 明确要求重试的瞬时服务端 message：`please retry`、`please try again`、`An unexpected error occurred`、`temporarily unavailable`、`unexpectedly high load`。
- 默认最多重试 5 次，可通过 `extension.js` 中的常量调整。
- 默认退避时间约为 1.5 秒、4 秒、9 秒、16 秒、25 秒，并加入少量随机抖动。
- 如果模型已经流出任何 chunk，则不再自动重试，避免重复执行工具调用、重复写文件或重复提交部分输出。
- 如果达到重试上限后仍失败，才保留原有错误路径，把错误展示给用户。
- retry 分支日志使用 `this.log.debug()`。该 logger 没有 `warn()` 方法，使用 `warn()` 会导致重试逻辑自身抛出 `this.log.warn is not a function`。

用户可以在 `extension.js` 中搜索并修改这些常量：

```js
KIRO_STREAM_RETRY_MAX_ATTEMPTS = 5;
KIRO_STREAM_RETRY_FOREVER = false;
KIRO_STREAM_RETRY_BACKOFF_MS = [1500, 4e3, 9e3, 16e3, 25e3];
```

含义如下：

- `KIRO_STREAM_RETRY_MAX_ATTEMPTS`：最大重试次数，默认 `5`；设为 `0` 表示不自动重试。
- `KIRO_STREAM_RETRY_FOREVER`：是否无限重试，默认 `false`；设为 `true` 后会忽略 `KIRO_STREAM_RETRY_MAX_ATTEMPTS`。
- `KIRO_STREAM_RETRY_BACKOFF_MS`：每次重试前等待的毫秒数；无限重试时，超过数组长度后会一直复用最后一个等待时间；如果误设为空数组，会回退到 25 秒。

这针对的是瞬时模型容量不足和网络抖动，不会重试鉴权失败、用量上限、prompt 过长、用户取消、参数校验失败等非瞬时错误。错误对象的 `cause` 链最多向下检查 10 层，避免异常对象循环导致卡住。

### 6. `getWorkspaceFiles()` 返回前按路径去重

`getWorkspaceFiles()` 现在会在返回前按 `path` 去重：

- 保留第一次出现的路径。
- 丢弃后续重复路径。
- 防止历史 `read_file` / `read_files` 调用不断放大 `[Steering] ExistingFiles`。

### 7. code problems formatter 跳过越界 diagnostics

code problems formatter 现在会跳过这些 diagnostics：

- 没有 `range`
- `start.line` 小于 0
- `end.line` 小于 0
- `start.line` 超过当前文档行数
- `end.line` 超过当前文档行数

这样单条 stale diagnostic 不会导致整段 code problems 格式化失败。

## 已执行验证

主 bundle 已通过语法检查：

```powershell
node --check "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
```

补丁后确认：

- metadata 安全写入 helper 仍存在。
- 受影响 task metadata 目录下没有残留 `codex-provider.meta.json*.tmp`。
- 最新日志中未再看到新的 `EPERM`、`EACCES` 或 `EBUSY` metadata 写入失败。
- Kiro 日志中出现多次成功写文件记录。

## 生效要求

Kiro 必须重新加载 extension host 后，这些改动才会生效。

可使用以下任一方式：

- 在 Kiro 中执行 `Developer: Reload Window`
- 完全重启 Kiro

在重新加载之前，已有 extension host 仍然运行内存里的旧 `extension.js`，因此日志可能继续出现旧行为。这不代表文件补丁没有写入。

Reload 后建议检查：

- `exthost.log` 不应再新增：

```text
INVALID decoration from extension 'kiro.kiroAgent': Error: The decoration is empty
```

- `Kiro Logs.log` 中的 `[Steering] ExistingFiles` 应该不再重复列出同一路径。
- metadata 写入不应再报告 `EPERM`、`EACCES` 或 `EBUSY`。

## 后续维护注意事项

- 这些是本地 bundle 文件补丁，不是 Kiro 官方源码改动。
- Kiro 应用更新可能覆盖 `extension.js`。
- 如果 Kiro 更新覆盖了补丁，参考本 README 重新比对；不适配时用用户自己提前备份的原版 `extension.js` 恢复。
- 每次编辑 bundle 后都要运行 `node --check`。
