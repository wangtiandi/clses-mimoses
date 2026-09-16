# 提示词：为 mimocode(mimo) 做会话查看器

> 粘到装有 **mimocode** 的机器上执行(可粘进 mimo 或 claude code 会话)。功能规格见同目录 [`SPEC.md`](SPEC.md)（若没有, 把它里面的命令/列/渲染/删除规格全做）。命令名建议叫 `mses`(mimo sessions) 避免和 claude 的 `clses` 混。

---

## 任务
写一个 `mses` 命令(单文件 Python, 纯标准库), 用表格列出本机 mimocode 的会话, 按序号恢复/查看/删除。mimo 把会话存在 **SQLite** 里(不是 jsonl), 所以读法和 Claude Code 不同。

## 数据源(已查明, 不用再发现)
- **主库**: `~/.local/share/mimocode/mimocode.db` (SQLite, WAL 模式, 有 `-shm`/`-wal` 边车)
- **只读打开** (并发安全): `sqlite3.connect("file:" + path + "?mode=ro", uri=True)`
- 关键表与列:
  - `session`: `id, project_id, parent_id, slug, directory, title, version, time_created, time_updated, time_compacting, time_archived, last_checkpoint_message_id`
    - **`title`** 就是首条提问 → 当"主题"
    - **`directory`** = 该会话原 cwd → 当"项目"列 + 恢复时 chdir
    - **`time_updated`** = 最后活动(**毫秒**时间戳, 显示要 /1000)
    - `time_archived` 非空 = 已归档(默认不列, 或单列)
  - `message`: `id, session_id, agent_id, time_created, time_updated, data`(`data` 是 JSON, 含 role/content)
    - **轮次** = `SELECT COUNT(*) FROM message WHERE session_id=?`(或只数 role=assistant)
    - **最后对话** = 该 session 最后一条 user 消息的文本(从 `message.data` JSON 里取, 按 `time_created` 倒序)
  - `part`: `id, message_id, session_id, time_created, time_updated, data`(更细的内容分片, 全文查看 `-x` 用)
  - `history_fts`: `part_id, session_id, message_id, project_id, kind, tool_name, body, time_created` —— **FTS5 全文索引表**, `body` 是文本
    - **`-g` 关键词过滤用它做真·全文搜索**: `SELECT DISTINCT session_id FROM history_fts WHERE body MATCH ?`(比 claude 那版只搜首/末句强)
  - `external_import`: 73 行, 记录从外部(如 Claude Code)导入的会话; `session_id` 关联到 session 表
- **提问历史**: `~/.local/state/mimocode/prompt-history.jsonl`(全局仅 prompt, 类似 claude 的 history.jsonl)
- **自带命令(可参考/复用)**:
  - `mimo session list` —— mimo 已能列会话, 先跑一遍看输出格式, 你的工具要比它表格更丰富(主题+最后对话+轮次+项目+时间+全文搜索+按序号恢复)
  - `mimo export [sessionID]` —— 导出单个会话为 JSON
  - `mimo import <file>` —— 从 JSON 导入

## 恢复
```python
os.chdir(directory)   # session.directory
os.execvp("mimo", ["mimo", "-s", session_id])   # -s / --session 续指定会话
```
`mimo -s <id>` 是官方续会话命令(`mimo --help` 里有)。

## 活跃判定(发现式, 三选一, 实测哪个准用哪个)
1. `~/.local/state/mimocode/locks/` 目录下有无该 session 的锁文件(本机当前为空, 说明无在跑)
2. `actor_registry` 表的 `status`/`session_id`(5 行, 含 status 字段)
3. `pgrep -f mimo` + 命令行里带该 session_id
任一命中即标 `*` 且拒绝删除。

## 删除 = 回收站(用 mimo 自带 export/import 做, 不是 rm DB 行)
mimo 存在 SQLite, 不能"移动文件", 但有 export/import 往返:
- **删除**: `mimo export <id>` 导出到 `~/.mses-trash/<id>/session.json` + meta.json(记 directory), 再 `mimo session delete <id>`(mimo 自带硬删) 删掉
- **恢复**: `mimo import ~/.mses-trash/<id>/session.json`(mimo 自带导入会重建)
- **列回收站**: 扫 `~/.mses-trash/` 下的 meta.json
这样删除可逆, 且复用 mimo 自己的 round-trip, 不直接动 DB。

## 注意
- mimo 会改写 `~/.claude/settings.json` 把 model 指向 `mimo-v2.5-pro`(本机有 `settings.mimo-backup.json` 备份)——脚本别碰这个文件
- `time_*` 字段是**毫秒**, 渲染要除 1000
- mimo 可能把 Claude Code 会话也导入(import-claude), 所以 session 表里会混入导入项, 按 `external_import` 区分来源(可选加一列"来源")
