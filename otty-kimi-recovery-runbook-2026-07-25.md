# 【恢复手册】Otty 终端 Tab 与 Kimi 会话的灾难恢复（2026-07-25）

> **这篇文章写给未来的 Kimi（或任何 AI 助手）**：当用户说"我的 tab 丢了 / 重启后会话没了，去看 kimi-os 仓库那篇恢复手册"时，你就是被指派来执行恢复的。请完整阅读本文，按"恢复操作手册"一节执行。
>
> 记录日期：**2026-07-25**（macOS，终端 Otty，CLI 为 Kimi Code CLI）

## 背景：今天发生了什么

2026-07-25 一天之内连续踩了这些坑，全部已解决，经验固化如下：

1. **重启 Mac 后**：Otty 把 tab 恢复成了普通 shell，kimi 聊天"不见了"——实际会话数据都在本地磁盘，没丢
2. **Otty 窗口被整个关闭后**：12 个 tab 全部消失，重开只带回 1 个 tab——Otty 的会话恢复**不保证**窗口关闭后的 tab 布局
3. **恢复后出现同名 tab**：`kimi -c`/`-S` 恢复会话时 kimi 会**分叉出会话副本**，新旧两个会话同名（都叫"卸载软件"），被当成两段对话分配给了两个 tab
4. **`otty-cli pane send-keys` 默认被禁用**：需要 `ipc-allow-send-keys = true`
5. **`otty-cli tab new --command` 实测命令没有执行**：建 tab 后需改用 `send-keys` 注入

## 核心事实（恢复的理论基础）

- **kimi 会话持久化在本地**：`~/.kimi-code/sessions/wd_<目录名>_<hash>/session_<uuid>/`
- **会话索引**：`~/.kimi-code/session_index.jsonl`，逐行 JSON，含 `workDir` 和 `sessionId`，追加写入（越靠后越新）
- **恢复命令**：`kimi -c`（续当前目录最近会话）、`kimi -S <sessionId>`（恢复指定会话）
- **恢复会分叉**：resume 会产生新会话副本，旧会话保留。因此同一个聊天内容可能存在多个 sessionId，恢复时注意去重
- **Otty 控制 CLI**：`/Applications/Otty.app/Contents/MacOS/otty-cli`，支持 `tab list/new`、`pane list/send-keys/capture`，均支持 `--json`
- **send-keys 开关**：`otty-cli config set ipc-allow-send-keys true && otty-cli config reload`
- **tab 只是窗口状态，会话才是数据**。tab 全丢也能照布局重建

## 恢复操作手册（给 AI 的执行步骤）

### 第 0 步：评估现状

```bash
OT=/Applications/Otty.app/Contents/MacOS/otty-cli
pgrep -x Otty                                    # Otty 是否在跑
$OT tab list --json                              # 现有 tab 及各自 cwd
$OT pane list --json                             # pane 及 process（非空=忙碌，不要碰）
```

- 忙碌 pane（`process` 非空，如正在跑 kimi）**一律跳过**，不要注入
- 现有的活跃 kimi 会话对应的 sessionId 要排除，不要分配给别的 tab

### 第 1 步：打开注入开关

```bash
$OT config set ipc-allow-send-keys true
$OT config reload
```

### 第 2 步：补齐缺失的 tab

对照下方"基准布局表"，用 `tab new` 补齐缺失的 tab（`--no-focus` 避免抢焦点）：

```bash
$OT tab new -q --no-focus --cwd "<目录>"            # 需要自定义标题加 --title "xxx"
```

> 注意：`tab new --command` 在 2026-07-25 的 Otty 版本上**实测不执行命令**，建完 tab 必须用 send-keys 注入。

### 第 3 步：注入会话恢复命令

```bash
$OT pane send-keys --pane <pane_id> "kimi -S <sessionId>" key:Enter
```

规则：同目录多个 tab 各分一个**不同**会话；单会话目录可用 `kimi -c`；分配前从候选池排除正在运行的会话。

### 第 4 步：验证

```bash
$OT pane list --json    # 每个 pane 的 process 应显示会话名
$OT pane capture --pane <id> | tail -5   # 应看到 kimi 输入框
```

## 基准布局表（2026-07-25 快照）

用户当时的 13 个 tab。sessionId 可能随时间新增/分叉，**优先以 `session_index.jsonl` 里该目录的最新会话为准**，此表用于核对 tab 结构和作为兜底：

| # | 标题/会话名 | 工作目录 | 当时分配的 sessionId |
|---|------------|---------|---------------------|
| 1 | （主目录会话） | `/Users/zzy` | `session_f6fc1d84-cf76-4c22-b255-fdaa23d6b862` |
| 2 | blog需求分析 | `/Users/zzy/code/codex/26-07-anji/zhenhua-blog` | `kimi -c`（目录最新） |
| 3 | 腾讯云ssh密钥 | `/Users/zzy/code/codex/26-07-anji/airport-kimi-1.0` | `session_a680b715-ec9d-44d5-835b-0b0e0fc3cd7b` |
| 4 | 111 （自定义标题） | `/Users/zzy` | `session_91a29d40-71bf-459e-b41c-0789617b3a2d` |
| 5 | airport部署到云服务器 | `/Users/zzy/code/codex/26-07-anji/airport-kimi-1.0` | `session_efce1709-62bc-4ffd-b540-d2f9a5ab01c6` |
| 6 | codex换皮肤 | `/Users/zzy` | `session_c09c5874-00a3-477a-9ec5-ecc7ba57f6e7` |
| 7 | 菲尔兹奖 | `/Users/zzy` | `session_7582de87-51a5-4789-ac05-70b0217a923b` |
| 8 | linuxdo备战站点 | `/Users/zzy` | `session_acfb03dd-be17-4a34-aa63-d5416ab2dd07` |
| 9 | 卸载软件（分叉副本） | `/Users/zzy` | `session_dc2b5c8b-7f49-4553-9467-975bd0e1c55b` |
| 10 | latex中德模版 | `/Users/zzy/code/kimicode/latex/izd-latex` | `kimi -c`（目录最新） |
| 11 | launch启动台 | `/Users/zzy` | `session_4fdf6e45-0b64-4fb2-9f84-ea713aba67c7` |
| 12 | cloudflare部署博客 | `/Users/zzy` | `session_e2466924-58c5-490d-914a-9a3db3fc52c8` |
| 13 | 卸载软件（当前活跃会话） | `/Users/zzy` | `session_16eee340-e78c-4718-800b-76dd1136528c` |

注意 #9 与 #13 是同一段对话的两个分叉副本（见"坑 3"），恢复时保留活跃的即可，旧副本可不恢复。

## 自动化方案

桌面有现成的双击脚本 `~/Desktop/恢复kimi会话.command`（教程见 [restore-kimi-tabs-script.md](restore-kimi-tabs-script.md)）：自动拉起 Otty、等待 tab 就绪、跳过忙碌 pane、按目录分配不同会话并注入。**但它只能恢复"还存在"的空闲 tab**；若 tab 整个丢失（坑 2 场景），需按本文第 2 步先补 tab 再恢复。

## 故障速查

| 症状 | 原因 | 处理 |
|------|------|------|
| 重启后 tab 里聊天没了 | tab 被恢复成普通 shell | `kimi -c` 或跑桌面脚本 |
| 整个窗口的 tab 全没了 | 窗口被关闭，Otty 未保存布局 | 按基准布局表 `tab new` 重建 + 注入 |
| 出现两个同名 tab | resume 分叉产生的会话副本 | 保留活跃的那个，旧副本 ⌘W 关掉 |
| `send-keys is disabled` | 注入开关未开 | `config set ipc-allow-send-keys true` + `reload` |
| `tab new --command` 没反应 | Otty 该版本不执行 command | 改用 `pane send-keys` 注入 |
| 注入后 tab 显示会话已存在/报错 | 同一会话被分配给两个 tab | 换该目录的另一个 sessionId 重试 |

## 相关笔记

- [kimi -c 基础恢复技巧](kimi-resume-session-after-restart.md)
- [otty-cli 批量恢复原理篇](otty-batch-restore-kimi-sessions.md)
- [一键恢复脚本教程](restore-kimi-tabs-script.md)
