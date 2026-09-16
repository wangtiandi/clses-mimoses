# clses-mimoses

会话查看器工具集。用表格列出、恢复、查看、删除本机 AI CLI 工具的对话会话。

| 工具 | 目标 | 数据源 |
|---|---|---|
| `clses` | Claude Code 会话 | `~/.claude/projects/<cwd编码>/*.jsonl` |
| `mimoses` | mimocode(mimo) 会话 | `~/.local/share/mimocode/mimocode.db`（SQLite） |

- Python 3.8+，**零第三方依赖**（纯标准库）
- 每个工具单文件、约 34 KB / 1000 行
- 同规格：表格 / 序号恢复 / 全文查看 / 回收站（export-import 往返）/ grep / 时间过滤 / JSON/CSV 输出

---

## 为什么

CLI 的内置 `/resume` 只给**自动生成的会话名**（Claude Code 那边是首条 prompt 派生的固定字段，且无法在对话内修改）。名字和实际内容经常对不上，看不出这个对话讲的是什么。

这两个工具绕开"名称"，直接读原始转录 / 数据库，把真实的首条用户提示（主题）+ 最后一条对话 + 完整 sessionID 摆出来。

---

## 安装

```bash
# 二选一，装谁拷谁：
chmod +x clses &&   mkdir -p ~/.local/bin && cp clses   ~/.local/bin/
chmod +x mimoses &&                          cp mimoses ~/.local/bin/

# 确保 ~/.local/bin 在 PATH：
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc   # 或 ~/.zshrc
```

无 Python 包、无编译、无运行时依赖。

---

## 用法

`clses` 和 `mimoses` 的接口完全一致（换掉工具名即可）：

```
<tool>                       列出当前项目(目录)的会话
<tool> N                     恢复第 N 个会话
<tool> -x N, --view N       展开第 N 个会话的全文对话
<tool> -d N, --delete N     删除第 N 个会话（移入回收站; -y 跳过确认）
<tool> --restore [N]        恢复第 N 个；省略 N 则列出回收站内容
<tool> --trash              列出回收站
<tool> -a, --all            列出所有项目的会话
<tool> -p PATH              列出指定目录的会话
<tool> -f, --full           加宽"主题/最后对话"列
<tool> -g PAT               按关键词过滤
<tool> -t RANGE             按时间过滤: today / 1h / 7d / 30d / 2w
<tool> --sort KEY           排序: time(默认) / oldest / size / turns
<tool> --json / --csv       以 JSON/CSV 输出
<tool> -y, --yes            删除时跳过确认
<tool> -h, --help           帮助
```

**表格列**：`# | Session ID | [项目] | 主题 | 最后对话 | 轮次 | 最后活动`。正在运行的会话在 `#` 列加 `*` 标记。

示例：

```
$ clses
当前项目 /path/to/proj 的会话（共 6 个，time 排序）:

┌─────┬──────────────────────────────────────┬─────────────────┬──────────────────────────────────┬──────┬─────────────┐
│   # │ Session ID                           │ 主题            │ 最后对话                         │ 轮次 │    最后活动 │
├─────┼──────────────────────────────────────┼─────────────────┼──────────────────────────────────┼──────┼─────────────┤
│  1* │ 8a50d886-4d36-44f7-b55d-5d53dfefa041 │ 某个主题示例    │ 最后一个用户提问片段…            │  392 │ 09-07 14:13 │
│   2 │ 1d59598a-b56a-418e-9e60-4c3d83f33baa │ 另一个主题      │ 另一个用户提问片段…              │  184 │ 09-04 18:37 │
```

---

## 数据源与安全性

| | clses | mimoses |
|---|---|---|
| 打开方式 | JSONL 文件（只读解析，不写） | SQLite `?mode=ro` |
| 删除 | `mimo export` / `claude` 无 export → 手工备份 | 用 `mimo export` 到 `~/.mimoses-trash/` 再 `mimo session delete` |
| 恢复 | 复制 JSONL 回原目录 | `mimo import` |

两个工具**不修改任何原始数据**，除了"删除"这个动作——删除永远先做备份再删，且活跃会话（有 PID 或锁）拒绝删除。

---

## 目录结构

```
.
├── clses                           # 权威脚本（Claude Code）
├── mimoses                         # 权威脚本（mimocode）
├── README.md                       # 本文件
├── LICENSE                         # MIT
├── prompts/
│   ├── SPEC.md                     # 通用规格（三个工具共享）
│   ├── build-claudecode.md         # 从零重建 clses 的提示词
│   ├── build-mimo.md               # 从零重建 mimoses 的提示词
│   └── build-opencode.md           # 计划中的第三个工具 openses 规格
└── docs/
    └── storage-and-retention.md    # Claude Code 本机存储 + 30 天清理机制分析
```

**为什么用 prompts/**：这些工具设计成"单文件 + 零依赖"，就是为了让读者能在别的机器/别的工具上从零重建。`prompts/SPEC.md` 是核心功能规格，`build-*.md` 是对应的引导提示词，粘到有对应工具的机器上执行即可。

---

## 已知限制

- `clses` 只在 macOS / Linux 上实测过。Windows 需要改 5 处 API（`os.kill`、`os.execvp`、`shutil.move` 遇到句柄锁、`isatty` VT 处理、cwd 编码规则）——**未验证**。
- `mimoses` 依赖 mimocode 的 SQLite schema（`session` / `message` / `history_fts` / `actor_registry`）。若 mimocode 改版，需按新 schema 调整 `load_sessions` / `view_session` / `is_active` 里的 SQL。
- `--grep` 用 SQLite FTS5（`mimoses`）或简单子串匹配（`clses`），不做正则。

---

## License

MIT. See [LICENSE](LICENSE).
