# kimi-os

Kimi Code CLI 的使用技巧与实战笔记。

## 笔记索引

| 笔记 | 内容 |
|------|------|
| [重启后恢复聊天记录：`kimi -c` 技巧](kimi-resume-session-after-restart.md) | 会话保存在本地，重启不丢；`kimi -c` 继续当前目录的上次会话，`kimi -S` 从历史会话中挑选；任何终端通用 |
| [一键批量恢复所有终端 Tab 的 Kimi 会话](otty-batch-restore-kimi-sessions.md) | Otty + otty-cli 实战：`tab list` 匹配会话、`pane send-keys` 批量注入，重启后 12 个 tab 一次全部恢复 |
| [一键恢复脚本（双击即用）](restore-kimi-tabs-script.md) | 成品 `.command` 脚本：放桌面双击即可批量恢复，含忙碌跳过、防撞车等保护设计 |
| [macOS 关闭 App 开机自启动完整指南](macos-disable-login-items.md) | 登录项 / LaunchAgents / BTM 三种自启动方式的排查与关闭，含防"复活"设置 |

## 快速参考

```bash
kimi -c          # 继续当前目录下最近的会话
kimi -S          # 交互式选择历史会话
kimi -S <会话ID>  # 恢复指定会话
```

会话数据存放在 `~/.kimi-code/sessions/`，按工作目录分桶（`wd_<目录名>_<hash>/`）。
