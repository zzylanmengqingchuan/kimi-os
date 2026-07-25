# macOS 关闭 App 开机自启动的完整指南（命令行实战）

macOS 上 App 的开机自启动有**好几种不同的注册方式**，只在系统设置里点开关经常关不干净。本文以 Steam、学习通、Warp、百度网盘、WPS 为例，讲清楚每种方式的排查和关闭方法。

## 自启动的三种注册方式

| 方式 | 典型表现 | 排查命令 |
|------|---------|---------|
| 登录项（Login Items） | 系统设置 → 通用 → 登录项与扩展 里可见 | `osascript` 列举（见下） |
| LaunchAgents 后台服务 | 开机后后台常驻进程 | `ls ~/Library/LaunchAgents` |
| BTM 现代后台项（含 App 内嵌助手） | 系统设置"允许在后台"列表 | `sfltool dumpbtm` |

## 第一步：盘点现状

列出传统登录项：

```bash
osascript -e 'tell application "System Events" to get the name of every login item'
```

查看 LaunchAgents 里有没有目标应用：

```bash
ls ~/Library/LaunchAgents /Library/LaunchAgents 2>/dev/null | grep -i -E "steam|baidu|wps|warp"
```

查看完整的后台项注册表（macOS 13+ 的 BTM）：

```bash
sfltool dumpbtm | grep -i -B2 -A6 -E "steam|baidu|wps|warp"
```

看 `Disposition` 一行：`enabled` 表示会自启动，`disabled` 表示已关闭。

## 第二步：删除登录项（可脚本化）

传统登录项可以用 AppleScript 批量删除：

```bash
for app in "学习通" "Warp" "Steam" "百度网盘"; do
  osascript -e "tell application \"System Events\" to delete login item \"$app\""
done
```

删完再跑一次列举命令验证。对大多数 App 类型的登录项，这一步同时也能让对应的 BTM 条目失效。

## 第三步：停用 LaunchAgents 后台服务

有些 App（如百度网盘的 `netdisk_service`、Steam 的 `steamclean`）通过 `~/Library/LaunchAgents` 下的 plist 开机常驻。**只删登录项管不到它们**，需要：

```bash
# 停掉当前运行的服务
launchctl bootout gui/$(id -u)/netdisk_service
launchctl bootout gui/$(id -u)/com.valvesoftware.steamclean

# 移走 plist（备份而非删除，想恢复随时移回）
mkdir -p ~/Library/LaunchAgents/_disabled_autostart
mv ~/Library/LaunchAgents/netdisk_service.plist ~/Library/LaunchAgents/_disabled_autostart/
mv ~/Library/LaunchAgents/com.valvesoftware.steamclean.plist ~/Library/LaunchAgents/_disabled_autostart/
```

plist 不存在后，开机时 launchd 就不会再拉起它们。`sfltool dumpbtm` 里的残留记录无害，系统会自行清理。

## 第四步：处理"只能手动关"的 BTM 内嵌助手

有些 App 把自启动助手打包在应用内部（如 WPS 的 `wpslaunchhelper`），这类条目**没有提供命令行关闭方式**，只能在 GUI 里关一次：

1. 打开 系统设置 → 通用 → 登录项与扩展（可用命令直达：`open "x-apple.systempreferences:com.apple.LoginItems-Settings.extension"`）
2. 在"**允许在后台**"列表找到对应应用（如 WPS Office）
3. 把开关关掉

## 第五步：防止 App"复活"自启动

很多 App 下次被打开时，会根据**应用内设置**重新注册开机启动。如果重启后发现又回来了，去应用内关掉：

- **Steam**：设置 → 界面 → 取消"开机时运行 Steam"
- **百度网盘**：设置 → 基本 → 取消"开机时启动"
- **Warp**：Settings → Features → Launch at login
- **WPS / 学习通**：各自设置里的"开机启动"选项

## 验证

全部设置完后重启一次，再跑第一步的三个排查命令，确认目标条目全部消失或变为 `disabled`。
