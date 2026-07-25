# 重启后一键恢复所有终端 Tab 的 Kimi 会话（Otty + kimi -c 实战）

**痛点**：Mac 重启后，终端（Otty）虽然能把之前的 tab 恢复出来，但每个 tab 都变成了普通的 shell——之前在各个目录里跟 Kimi Code CLI 聊到一半的对话，看起来"全没了"。

**结论**：会话其实一直都在本地。配合 kimi 自带的恢复参数和 Otty 的控制 CLI，可以做到**一条脚本把所有 tab 的会话全部批量恢复**。从此重启电脑不用再担心 tab 丢失、会话流失。

## 第一步：理解两套独立的恢复机制

| 恢复什么 | 谁负责 | 方式 |
|---------|--------|------|
| 终端窗口 / tab / 工作目录 | 终端 App（Otty、Warp 等） | 终端自己的会话恢复 |
| Kimi 聊天内容 | Kimi Code CLI 自己 | `kimi -c` / `kimi -S` |

重启后 tab 还在、聊天"没了"——其实是 tab 被恢复成了普通 shell。聊天数据完好地存在本地（`~/.kimi-code/sessions/`），捞回来就行。

## 第二步：单 tab 手动恢复

```bash
kimi -c          # 继续【当前目录】下最近的会话
kimi -S          # 列出历史会话，交互式选择
kimi -S <会话ID>  # 直接恢复指定会话
```

注意点：

- `-c` 是 `--continue` 的缩写，**按工作目录匹配**——要先 `cd` 到当时聊天的目录再执行
- 会话数据按目录分桶存放在 `~/.kimi-code/sessions/wd_<目录名>_<hash>/` 下，每个桶里是该目录的历史会话

手动一个个敲已经够用，但十几个 tab 时还是麻烦。下面进入自动化部分。

## 第三步：用 otty-cli 查看和控制 tab（Otty 专属）

Otty 自带控制 CLI（在 App 包内）：

```bash
OT=/Applications/Otty.app/Contents/MacOS/otty-cli
```

列出所有 tab（含每个 tab 的工作目录）：

```bash
$OT tab list --json
# 输出每个 tab 的 id、cwd、title、是否 active
```

向指定 pane 发送按键：

```bash
$OT pane send-keys --pane <pane_id> "kimi -c" key:Enter
```

**前提：Otty 默认禁止远程按键注入**，需要先打开开关并重载配置：

```bash
$OT config set ipc-allow-send-keys true
$OT config reload
```

> 安全提示：开启后本机程序都能向终端注入按键。用完可以 `$OT config set ipc-allow-send-keys false` 再 reload 关掉。

## 第四步：建立「tab → 会话」的映射

核心思路：

1. `tab list --json` 拿到每个 tab 的 `cwd`
2. 遍历 `~/.kimi-code/sessions/wd_*`，找到每个目录有哪些历史会话（按修改时间排序，新的优先）
3. 对应规则：
   - tab 的目录**只有一个会话** → 直接在该 tab 执行 `kimi -c`
   - **多个 tab 同目录**（比如两个 tab 都在同一项目）→ 给每个 tab 用 `kimi -S <会话ID>` 分配**不同的**会话，避免抢同一个
   - **当前正在使用的 tab 跳过**，它对应的会话也不要分配给别人

查看某目录有哪些会话：

```bash
ls -t ~/.kimi-code/sessions/wd_<目录名>_<hash>/
# 按时间从新到旧列出 session_<uuid>
```

## 第五步：批量注入，一键恢复

对映射好的每个 pane 执行（示例）：

```bash
OT=/Applications/Otty.app/Contents/MacOS/otty-cli

# 单会话目录：直接 kimi -c
$OT pane send-keys --pane p_aaa "kimi -c" key:Enter

# 多 tab 同目录：各分一个会话
$OT pane send-keys --pane p_bbb "kimi -S session_a680b715-..." key:Enter
$OT pane send-keys --pane p_ccc "kimi -S session_efce1709-..." key:Enter
```

全部发完后，用 `pane capture` 验证每个 tab 都出现了 kimi 的输入框：

```bash
$OT pane capture --pane p_aaa | tail -4
```

看到 `│ >` 的输入提示符，说明该 tab 的会话已经恢复成功。

## 实际效果

一次操作把 12 个空闲 tab 全部恢复到了各自的 kimi 会话：

- 项目目录 tab（博客、LaTeX 等）→ `kimi -c` 恢复最新会话
- 同目录的多个 tab → 各分一个不同的历史会话
- 主目录的零散 tab → 按时间远近分配不同会话

恢复后逐个检查，分配不理想的 tab 直接 ⌘W 关掉即可，没有任何副作用。

## 总结

- **重启不丢会话**：kimi 的聊天数据持久化在本地，`kimi -c` / `kimi -S` 随时捞回
- **tab 恢复和会话恢复是两件事**：终端管窗口，kimi 管聊天
- **Otty 用户可更进一步**：`otty-cli` 的 `tab list` + `pane send-keys` 能把"逐个敲恢复命令"变成一键批量操作
- 这个思路与终端无关的部分（`kimi -c` / `-S`）在 Warp、iTerm2、系统终端里同样适用
