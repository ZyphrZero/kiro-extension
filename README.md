# Kiro Agent Extension 本地补丁

这是一个针对 Kiro agent 扩展 bundle 的本地补丁包。仓库里的 `extension.js` 不是官方源码，而是已打补丁的本地 bundle，主要用于缓解 Windows 下 Kiro spec/task 执行时的写入失败、流式写入中断、模型高负载失败和日志噪声。

> Kiro 更新可能覆盖此补丁。替换前务必备份原版 `extension.js`。

English notes: [README_EN.md](README_EN.md)

详细技术说明：[docs/TECHNICAL_DETAILS.md](docs/TECHNICAL_DETAILS.md) / [English](docs/TECHNICAL_DETAILS_EN.md)

## 解决的问题

适用于遇到以下问题的 Windows Kiro 用户：

- metadata 写入失败：`EPERM`、`EBUSY`、`EACCES`
- 文件写入中断：`Operation was aborted by user or system`、`Stream error`
- spec/task 偶发失败：`Failed to invoke Spec Task Execution`
- 模型高负载、限流或网络短暂抖动导致任务直接失败
- extension host 反复提示：`The decoration is empty`
- `[Steering] ExistingFiles` 文件列表重复膨胀
- stale diagnostics 导致 code problems 格式化失败

如果你的 Kiro 没有这些问题，不建议替换官方 bundle。

## 文件

| 文件 | 说明 |
| --- | --- |
| `extension.js` | 已打补丁的 Kiro agent 扩展 bundle |
| `README.md` | 中文说明 |
| `README_EN.md` | 英文说明 |
| `docs/TECHNICAL_DETAILS.md` | 中文详细技术说明 |
| `docs/TECHNICAL_DETAILS_EN.md` | 英文详细技术说明 |

## 快速安装

关闭 Kiro 后执行以下命令。路径请按你的 Kiro 安装目录调整：

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"

# 1. 备份原版
Copy-Item $target "$target.original-backup"

# 2. 检查补丁文件语法
node --check .\extension.js

# 3. 替换 bundle
Copy-Item .\extension.js $target -Force
```

替换后重新加载 Kiro：

- 在 Kiro 中执行 `Developer: Reload Window`
- 或完全退出并重新启动 Kiro

如果复制时报权限错误，先完全关闭 Kiro；仍失败时，用管理员 PowerShell 执行。

## 验证

安装后建议做一次快速检查：

```powershell
node --check "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"

Select-String -Path "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js" -Pattern "KIRO_STREAM_RETRY_MAX_ATTEMPTS","KIRO_STREAM_RETRY_FOREVER","KIRO_STREAM_RETRY_BACKOFF_MS"
```

预期结果：

- `node --check` 无报错。
- 能搜索到 `KIRO_STREAM_RETRY_*` 常量。
- Kiro reload 后，日志不再新增 `The decoration is empty`。
- metadata 写入不再新增 `EPERM`、`EACCES`、`EBUSY`。
- 模型高负载、限流、网络瞬断类错误会先有限重试。

## 回滚

如果补丁不适配当前 Kiro 版本，恢复备份并重启 Kiro：

```powershell
$target = "D:\Kiro\resources\app\extensions\kiro.kiro-agent\dist\extension.js"
Copy-Item "$target.original-backup" $target -Force
```

## 补丁内容

| 类别 | 补丁行为 |
| --- | --- |
| metadata 写入 | 对 Windows 瞬时 busy 错误做重试；必要时 fallback 直接写目标文件 |
| 写入工具 | 引导 `fs_write` / `fs_append` 使用小块写入，避免超大流式参数被 abort |
| `tasks.md` 生成 | 限制一次性全量写入，首段 `fs_write`，后续 `fs_append` |
| FileDecoration | 无装饰时返回 `undefined`，避免空 decoration 警告 |
| 模型流错误 | 对高负载、临时限流、网络抖动做有限自动重试 |
| ExistingFiles | `getWorkspaceFiles()` 返回前按路径去重 |
| code problems | 跳过无效或越界 diagnostics，避免整体格式化失败 |

默认重试配置在 `extension.js` 中：

```js
KIRO_STREAM_RETRY_MAX_ATTEMPTS = 5;
KIRO_STREAM_RETRY_FOREVER = false;
KIRO_STREAM_RETRY_BACKOFF_MS = [1500, 4e3, 9e3, 16e3, 25e3];
```

不建议默认开启无限重试。服务长时间高负载时，无限重试会让任务长期占用执行状态。

## 已知限制

- 不处理历史状态问题：`Cannot restore file ... original checkpoint missing for append action`
- 不处理 Kiro bundle 元数据警告：`No bundle location found for extension kiro.kiroAgent`
- 不重试非瞬时错误：鉴权失败、用量上限、prompt 过长、用户取消、参数校验失败等
- Kiro 应用更新后可能需要重新替换补丁

## 维护建议

- 每次替换前先备份官方 `extension.js`。
- 每次修改 bundle 后运行 `node --check`。
- Kiro 更新后先确认补丁是否被覆盖。
- 只使用你信任来源的 `extension.js`。

## 社区支持

欢迎到 [linux.do](https://linux.do/) 交流、分享和反馈。

---

<div align="center">
  <p>Made with ❤️ by <a href="https://github.com/ZyphrZero">ZyphrZero</a></p>
</div>
