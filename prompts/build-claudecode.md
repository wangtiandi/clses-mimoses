# 提示词：为 Claude Code 做会话查看器

> 粘到目标机的 **Claude Code** 会话里执行。功能规格见同目录 [`SPEC.md`](SPEC.md)（若没有, 把它里面的命令/列/渲染/删除规格全做）。

---

## 任务
写一个 `clses` 命令(单文件 Python, 纯标准库), 用表格列出本机 Claude Code 的会话, 按序号恢复/查看/删除。先实现 [`SPEC.md`](SPEC.md) 的全部功能, 数据源与恢复按下面来。

## 数据源(已查明, 不用再发现)
- **转录目录**: `~/.claude/projects/<cwd编码>/*.jsonl`
- **cwd 编码规则**: `re.sub(r"[^a-zA-Z0-9]", "-", cwd)` —— 所有非字母数字字符→`-`(`.`、`_` 也换)。例: `/home/qd/.claude` → `-home-qd--claude`; `/data/bysy_baobiao` → `-data-bysy-baobiao`
- **JSONL 每行一个对象**, 关键字段: `type`(`user`/`assistant`/`tool_result`/`last-prompt` 等)、`message.content`、`timestamp`、`sessionId`、`cwd`、`model`
- **主题** = 首条真实 user 提示(跳过 `isMeta`、`<command-name>`/`<local-command*>` 开头、纯 tool_result 的 user 消息)
- **最后对话** = 优先取 `type=last-prompt` 的 `lastPrompt` 字段, 没有则回退扫最后一条真实 user 消息
- **轮次** = `type=assistant` 的行数
- **项目列** = 该会话的 cwd basename(可从 jsonl 里 `cwd` 字段或目录名反推)
- **提问历史**(全局, 仅 prompt): `~/.claude/history.jsonl`, 不在 30 天清理内, 可作交叉参考
- **元数据(幽灵记录)**: `~/.claude.json`, 含 lastSessionId/lastCost 等, 转录删了元数据还在

## 恢复
```python
os.chdir(target_cwd)   # 该会话原 cwd
os.execvp("claude", ["claude", "--resume", session_id])
```
`claude --resume <完整UUID>` 是官方恢复命令。

## 活跃判定
- `~/.claude/sessions/<pid>.json` 列出活跃会话
- 用 `os.kill(pid, 0)` 校验 PID 仍存活(防 claude 崩溃残留文件误判), 抛 PermissionError 也算活着
- 存活 → 标 `*` 且拒绝删除

## 删除=回收站
移 `~/.claude/projects/<编码>/<sid>.jsonl`(连同附件目录)到 `~/.clses-trash/<epoch>_<sid>/` + meta.json; `--restore` 移回。

---

## Windows 变体(同一脚本加 win32 分支, Linux 行为不变)
直接复制 Linux 版到 Windows **不能跑**, 有 5 处要改(详见 [`../docs/windows-port.md`](../docs/windows-port.md)):
1. `os.kill(pid,0)` → `ctypes.windll.kernel32.OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION=0x1000)` + `GetExitCodeProcess==259`(STILL_ACTIVE)
2. `os.execvp` → `subprocess.call([shutil.which("claude") or "claude.exe", "--resume", sid], cwd=target_cwd)`(Python spawn 不认 npm 的 `.cmd` shim)
3. ANSI 色 → 启动加 `os.system("")` 启用 VT
4. `shutil.move` 删文件 → claude 正开着 jsonl 句柄时 Windows 拒绝移动, 必须先过活跃校验
5. `encode_cwd` → 逻辑不变, 但 Windows cwd 是 `C:\Users\qd\proj`, **先 `dir %USERPROFILE%\.claude\projects` 拿实际目录名对照**, 别猜驱动器字母/`\` 的处理

**Windows 验证清单**: 列出 / `-x` 全文 / `clses 1` 真能拉起 claude / `-d`+`--restore` 往返(文件锁重点) / 活跃 `*`+拒删 / `--json` 被 jq 消费。

## ⚠ 不要把脚本里读到的 `~/.claude/settings.json` 内容回传给我(含 API 凭据)
