# kimi-os

Kimi Code CLI 的使用技巧与实战笔记。

## 笔记索引

| 笔记 | 内容 |
|------|------|
| [重启后恢复聊天记录：`kimi -c` 技巧](kimi-resume-session-after-restart.md) | 会话保存在本地，重启不丢；`kimi -c` 继续当前目录的上次会话，`kimi -S` 从历史会话中挑选；任何终端通用 |
| [一键批量恢复所有终端 Tab 的 Kimi 会话](otty-batch-restore-kimi-sessions.md) | Otty + otty-cli 实战：`tab list` 匹配会话、`pane send-keys` 批量注入，重启后 12 个 tab 一次全部恢复 |
| [一键恢复脚本（双击即用）](restore-kimi-tabs-script.md) | 成品 `.command` 脚本：放桌面双击即可批量恢复，含忙碌跳过、防撞车等保护设计 |
| [macOS 关闭 App 开机自启动完整指南](macos-disable-login-items.md) | 登录项 / LaunchAgents / BTM 三种自启动方式的排查与关闭，含防"复活"设置 |
| [【恢复手册】Tab 与 Kimi 会话灾难恢复（2026-07-25）](otty-kimi-recovery-runbook-2026-07-25.md) | 给 AI 看的恢复手册：13 个 tab 的基准布局 + 会话 ID 快照、全部踩坑记录与故障速查 |
| [腾讯云 SSH 密钥免密登录：两种方式的区别](tencent-cloud-ssh-key-two-methods.md) | 本机生成 vs 控制台生成对比；私钥创建时浏览器静默自动下载、仅一次机会的大坑；网页终端 ≠ 本机 ssh |
| [国内服务器接 LinuxDo 登录失败：备用端点](linuxdo-connect-domestic-server-backup-endpoints.md) | `connect.linux.do` 被墙导致服务端换 token 失败；把服务端两个端点换成 `connect.linuxdo.org` 即可，授权页不用动 |
| [复刻 zhenhua.lu 博客并部署到 Cloudflare Pages](replicate-zhenhua-lu-blog-to-cloudflare-pages.md) | 克隆 MIT 开源博客源码换个人数据，Next.js 静态导出 + Pages 免备案部署，含 OAuth token 无 DNS 权限、fake-ip 干扰等 5 个坑 |

## 快速参考

```bash
kimi -c          # 继续当前目录下最近的会话
kimi -S          # 交互式选择历史会话
kimi -S <会话ID>  # 恢复指定会话
```

会话数据存放在 `~/.kimi-code/sessions/`，按工作目录分桶（`wd_<目录名>_<hash>/`）。
