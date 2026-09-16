# 提示词：为 opencode 做会话查看器

> 粘到装有 **opencode** 的机器上执行。功能规格见同目录 [`SPEC.md`](SPEC.md)（若没有, 把它里面的命令/列/渲染/删除规格全做）。命令名建议 `oses`(opencode sessions)。

---

## 任务
写一个 `oses` 命令(单文件 Python, 纯标准库), 用表格列出本机 opencode 的会话, 按序号恢复/查看/删除。

## ⚠ 先做发现(opencode 跨版本存储变了, 不能写死)
opencode 本机可能没装, 所以**第一步是摸清存储**, 再写代码:

```bash
which opencode
opencode --help | grep -iE 'resume|session|list|continue'
opencode resume --help 2>/dev/null     # 找恢复命令的确切写法
# 定位存储
find ~/.local/share/opencode ~/.config/opencode ~/Library/Application\ Support/opencode "$LOCALAPPDATA/opencode" -maxdepth 4 2>/dev/null | head -40
```

## 已知的两条可能路径(实测确认是哪条)
opencode 经历过两种存储, 新旧不一样:

**A. 每实体一个 JSON 文件(较旧版本)**
- 根目录: `~/.local/share/opencode/storage/`
- 子目录: `session/<id>.json`、`message/<id>.json`、`branch/`、`snapshot/`
- `session/*.json` 字段(参考): `id, title, time(创建, ms), cwd, parentID, shareURL, modelID`
- `message/*.json` 字段(参考): `id, sessionID, role(user/assistant), time, parts`(parts 是内容数组, 含 text/tool 调用)
  - **主题** = 该 session 最早一条 user message 的文本
  - **最后对话** = 该 session 最后一条 user message 的文本
  - **轮次** = role=assistant 的 message 数
  - **项目列** = session.cwd 的 basename
- 这种结构直接 `glob` + `json.load` 读, 像读 Claude Code 的 jsonl 一样

**B. SQLite(较新版本)**
- 根目录可能有 `~/.local/share/opencode/opencode.db` 或 storage 下的 `*.db`
- 若是 SQLite → 照 [`build-mimo.md`](build-mimo.md) 的套路: `?mode=ro` 只读打开, 查 `session`/`message` 表, FTS 表做 `-g`

## 恢复(发现后填实)
opencode 的恢复命令**未确认**, 候选:
- `opencode resume` (交互 TUI 选择)
- `opencode resume <session-id>` (传 id)
- `opencode run --session <id>` 或 `-c`(非交互续跑)
跑 `opencode resume --help` 确认后, 模板:
```python
os.chdir(session_cwd)
os.execvp("opencode", ["opencode", "resume", session_id])   # 按实际 subcommand/flag 改
```

## 活跃判定(发现式)
- 找 opencode 的锁/状态文件(常在 `~/.local/state/opencode/` 或 storage 下)
- 或 `pgrep -f opencode` + 命令行带 session_id
- 或 session JSON 里有 `time` 比对"最近 N 分钟"兜底
任一命中 → 标 `*` + 拒删。

## 删除 = 回收站
- 若是 **JSON 文件存储**: 移 `session/<id>.json`(+ 该 session 的 message 文件)到 `~/.oses-trash/<id>/` + meta.json(记 cwd), `--restore` 移回。和 Claude Code 版完全一样。
- 若是 **SQLite**: 不直接删行, 改用"导出→硬删→导入"往返(opencode 若有 export/import 命令就用; 没有就 `sqlite3` 备份该 session 的所有行到 trash 目录的 `.sql`/`.json`, 再从主库 DELETE, 恢复时 INSERT 回去)。

## 注意
- opencode 是 Go 写的, 时间戳可能是**毫秒**或 ISO 字符串, 实测看哪种, 渲染时统一
- 跨平台路径: Linux `~/.local/share/opencode`, Windows `%LOCALAPPDATA%\opencode`, Mac `~/Library/Application Support/opencode` —— 全用 `os.path.expanduser` / 环境变量, 别写死
- 不要把读到的任何含 token 的配置文件内容回传给我
