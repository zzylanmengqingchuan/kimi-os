# 重启后一键恢复所有 Kimi 会话：可双击运行的恢复脚本

配套笔记：[一键批量恢复所有终端 Tab 的 Kimi 会话（原理篇）](otty-batch-restore-kimi-sessions.md)

上文讲了用 `otty-cli` 批量恢复的原理。本文给出一个**放在桌面、双击即用**的成品脚本，并说明它的保护设计。

## 使用方法

1. 把下面的脚本保存为 `恢复kimi会话.command`（比如放在桌面）
2. 赋予可执行权限：`chmod +x ~/Desktop/恢复kimi会话.command`
3. 以后每次重启电脑后，**双击它**即可：脚本会自动拉起 Otty（如果没开）、等 tab 恢复完、给每个空闲 tab 分配并恢复各自的 kimi 会话

## 脚本全文

```bash
#!/bin/bash
# 恢复 Otty 所有空闲 tab 的 Kimi 会话（重启后双击运行）
#
# 安全性说明：
#   - 本脚本只向"空闲"的 pane 发送文本按键（kimi -S ...），
#     不创建、不关闭、不删除任何 tab/pane，不会破坏现有标签页
#   - 正在运行程序的 pane 一律跳过，不会误输入

OT="/Applications/Otty.app/Contents/MacOS/otty-cli"

if [ ! -x "$OT" ]; then
  echo "错误：未找到 Otty CLI（$OT）"
  read -n 1 -s -r -p "按任意键关闭..."
  exit 1
fi

# Otty 没在跑就先拉起来
if ! pgrep -x Otty >/dev/null 2>&1; then
  echo "Otty 未运行，正在启动..."
  open -a Otty
fi

# 确保允许远程按键注入（已开启则无副作用）
"$OT" config set ipc-allow-send-keys true -q
"$OT" config reload -q

python3 - <<'EOF'
import json, subprocess, collections, os, sys, time

OT = "/Applications/Otty.app/Contents/MacOS/otty-cli"
INDEX = os.path.expanduser("~/.kimi-code/session_index.jsonl")

def list_panes():
    out = subprocess.run([OT, "pane", "list", "--json"],
                         capture_output=True, text=True, check=True).stdout
    return json.loads(out)["data"]

# 1. 等待 Otty 的 tab 恢复完毕（刚启动时可能还没准备好）
panes = []
for attempt in range(10):
    try:
        panes = list_panes()
        if panes:
            break
    except Exception:
        pass
    time.sleep(2)
if not panes:
    print("错误：等了 20 秒仍无法获取 Otty 的 tab 列表，请确认 Otty 已正常打开")
    sys.exit(1)

# 2. 建立 工作目录 -> 会话列表（新的在前，去重）
wd_sessions = collections.defaultdict(list)
try:
    with open(INDEX) as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                d = json.loads(line)
            except json.JSONDecodeError:
                continue
            wd, sid = d.get("workDir"), d.get("sessionId")
            if wd and sid:
                wd_sessions[wd].append(sid)
except FileNotFoundError:
    print("错误：未找到 kimi 会话索引", INDEX)
    sys.exit(1)

for wd, ids in wd_sessions.items():
    seen, uniq = set(), []
    for s in reversed(ids):  # 索引是追加写的，倒序 = 最新在前
        if s not in seen:
            seen.add(s)
            uniq.append(s)
    wd_sessions[wd] = uniq

# 3. 忙碌的 pane：跳过；并消费掉该目录最新的一个会话，
#    避免把它（很可能就是这个 pane 正在跑的会话）再分配给别的 tab
for p in panes:
    proc = (p.get("process") or "").strip()
    cwd = p.get("cwd") or ""
    if proc and wd_sessions.get(cwd):
        wd_sessions[cwd].pop(0)

# 4. 遍历 pane，逐个恢复
restored, skipped_busy, skipped_nosession, failed = 0, 0, 0, 0
for p in panes:
    pid = p.get("id")
    cwd = p.get("cwd") or ""
    proc = (p.get("process") or "").strip()
    if proc:
        print(f"跳过 {pid}（正在运行: {proc[:40]}）")
        skipped_busy += 1
        continue
    sessions = wd_sessions.get(cwd)
    if not sessions:
        print(f"跳过 {pid}（{cwd}：没有可分配的历史会话）")
        skipped_nosession += 1
        continue
    sid = sessions.pop(0)  # 每个 tab 分配一个不同的会话
    r = subprocess.run([OT, "pane", "send-keys", "--pane", pid,
                        f"kimi -S {sid}", "key:Enter"],
                       capture_output=True, text=True)
    if r.returncode == 0:
        print(f"恢复 {pid}（{cwd}）-> {sid[:24]}...")
        restored += 1
    else:
        print(f"失败 {pid}: {r.stderr.strip() or r.stdout.strip()}")
        failed += 1

print(f"\n完成：恢复 {restored} 个，跳过忙碌 {skipped_busy} 个，无会话 {skipped_nosession} 个，失败 {failed} 个")
EOF

echo
read -n 1 -s -r -p "按任意键关闭窗口..."
```

## 保护设计（为什么它不会"把 tab 整没"）

- **只"打字"，不碰 tab**：全程只调用 `pane send-keys` 发送文本，没有创建/关闭/删除任何 tab 的操作
- **忙碌 tab 不碰**：`process` 非空（正在跑 kimi、ssh、vim 等）的 pane 一律跳过
- **会话不撞车**：同目录的多个 tab 各分配不同会话；忙碌 tab 正在用的会话会被"占用"，不再分配给别人
- **没聊过天的目录**：跳过，保持普通 shell
- **Otty 没开 / tab 还没恢复完**：自动拉起 Otty 并重试等待最多 20 秒
- **单个失败不影响整体**：某个 tab 注入失败会打印出来，其余继续
- **重复执行安全**：已恢复的 tab 在第二轮全部会被识别为忙碌而跳过

## 前置条件

- 终端是 Otty（其他终端用户直接用 `kimi -c` / `kimi -S` 手动恢复即可，见[基础篇](kimi-resume-session-after-restart.md)）
- 脚本依赖 `python3`（macOS 自带）
- 脚本会自动打开 Otty 的 `ipc-allow-send-keys` 开关；介意的话用完后可手动关闭：
  `/Applications/Otty.app/Contents/MacOS/otty-cli config set ipc-allow-send-keys false`
