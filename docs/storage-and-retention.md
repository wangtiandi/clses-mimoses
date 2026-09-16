# Claude Code 本机存储分布与保留期

排查「我的对话在哪 / 能不能恢复 / 为什么不见了」时的参考。适用机器：本机（用户 `qd`，`CLAUDE_CONFIG_DIR` 未设，`~/.claude` 就是全部本地存储）。

---

## 四个存储位置（按保留时长，越往下越持久）

### 1. `~/.claude/projects/<cwd编码>/*.jsonl` — 完整对话转录

user + assistant + 工具调用全部在里面。**⚠ 有 ~30 天自动清理（见下）**。清理只删 `.jsonl`，**保留同目录的 `memory/` 子目录**。

- 编码规则见 [README.md](../README.md#1-数据源与-cwd-编码规则最容易搞错的地方)
- 用户用 `clses -d` 删的 → 进回收站，可恢复

### 2. `~/.claude/.clses-trash/` — clses 回收站

用 `clses -d` 删的会话在这里，`clses --restore` 可恢复。**不受 30 天清理影响**（自己掌控）。

### 3. `~/.claude/history.jsonl` — 全局提问历史

**只有 prompt，没有 Claude 的回答**。每条：`{display, pastedContents, timestamp, project, sessionId}`，含斜杠命令（`/init` `/compact` 等）。

**不在 30 天清理范围内**：实测 3381 条 / 97 天 / 288 个 sessionId 连续无断，远超 30 天。

**但没有备份**，`rm` 掉就没了。

### 4. `~/.claude.json` — 每项目元数据（"幽灵记录"）

`lastSessionId` / `lastCost` / `lastTotalInputTokens` / `lastDuration` 等。**转录删了，元数据还在**——所以可以据此发现「哪些项目曾经有会话但转录已丢」，这是下面盘点里 54 个丢失转录的主要线索。

备份在 `~/.claude/backups/.claude.json.backup.*`，**但备份会轮转**，实测只留最近约 5 份。

---

## ~30 天自动清理：取证过程

这是「为什么之前的会话丢了」的答案。三个证据互相印证：

1. **时间线**：4 个「只剩 `memory/`、无转录」的项目，转录都在 `memory/` 写入后 **31 / 35 / 31 / 31 天**被删（目录 mtime = 删除时间，`memory/` 保留）
2. **年龄分布**：30 个存活转录的年龄**全部 < 28 天**；**> 30 天桶为 0 条**。如果存在超 30 天仍存活的转录，就证伪了自动清理
3. **排除人为**：`bash_history` 无删除命令、无 cron、无清理脚本

**教训：「没配置项」≠「没这行为」。** 内置行为在 `--help` 和 settings 文档里都查不到开关，得靠数据的时间分布反推。

---

## 配置项：`cleanupPeriodDays`

| | |
|---|---|
| 位置 | `~/.claude/settings.json` 顶层键 |
| 类型 | 正整数，最小 1，**默认 30** |
| 本机当前值 | **3650**（≈10 年，等价基本永久） |

官方 schema 文案（从二进制里挖出来）：

> "Number of days to retain chat transcripts before automatic cleanup (default: 30). Minimum 1. Use a large value for long retention; use --no-session-persistence to disable transcript writes entirely."

官方错误文案（把值设成 0 时）：

> "cleanupPeriodDays must be at least 1. To keep transcripts for a long time, set a large number (e.g. 3650 for ~10 years). To disable transcript writes entirely, remove this setting and use the --no-session-persistence CLI flag or the SDK persistSession:false option instead. (0 is rejected because it previously silently disabled all transcript writes, which users setting it to mean 'never clean up' did not expect.)"

**要点**

- `3650` 是**官方错误文案里给的示例值**，不是我猜的
- `0` 被拒绝；历史上 `0` 会**静默禁用所有转录写入**（想"永不清理"的人踩坑）
- 该键在 `--help` 和 settings 文档里**都不暴露**，是 `strings` 挖二进制（`@anthropic-ai/claude-code/bin/claude.exe`）找到的
- **改这个只影响未来的清理，已删的转录不会回来**
- 反向操作（完全不存转录）：CLI 加 `--no-session-persistence`，或 SDK 设 `persistSession: false`

### 改之前先检查有没有被覆盖

`settings.local.json` 优先级更高，改之前要确认它不含该键：

```bash
python3 -c "
import json,os
for f in ['settings.json','settings.local.json']:
    p=os.path.expanduser('~/.claude/'+f)
    d=json.load(open(p))
    print(f, '->', 'cleanupPeriodDays' in d, d.get('cleanupPeriodDays'))
"
```

本机实测：`settings.local.json` 顶层只有 `permissions` 和 `disabledMcpjsonServers`，不覆盖。

---

## 2026-09-07 盘点（本机）

| 类别 | 数量 |
|---|---|
| 转录可见 | 30 条（全部 < 28 天） |
| 回收站 | 34 条（31 条有内容，可恢复） |
| **真实对话转录已丢** | **54 个**（≥4 条提问） |
| "1 条提问无回复"空壳 | 142 个 |

**已丢的 54 个里，16 个是"整项目无转录"**，含：

| 项目 | 记录 |
|---|---|
| `/data/vm-system` | 427 万 token / $309 / 70 小时 |
| `/data/FeishuDataBridge` | $69 |
| `/data/pgsql-monitor` | $46 |
| `/mnt/hgfs/ifly_demo` | $34 |
| `/data/pgsql` | $28 |
| **合计** | **约 667 万 token / $575** |

这些的**提问仍在 `history.jsonl`**（能看问了什么，看不到答了什么）。

142 个空壳不算丢失——claude 从来没存过转录（你输入后取消 / 退出）。

---

## 教训与建议

1. **原来有个 30 天悬崖**：重要会话要么 30 天内备份，要么保持持续活跃（活跃不被删）。现在已经把 `cleanupPeriodDays` 设到 3650，悬崖取消了，但**过去丢的补不回来**。
2. 如果还想更保险，定期备份：
   ```bash
   rsync -a ~/.claude/projects/ /backup/claude-projects/
   ```
   `memory/` 子目录也在里面，一并带走。
3. 判断"转录在哪"用 [`clses`](../README.md)；判断"曾经有过但转录没了"用 `~/.claude.json` 的元数据 + `history.jsonl` 的提问记录交叉比对。
4. **云端不在本机**：`claude --cloud`、网页 claude.ai/code、IDE 插件的转录和提问都在 Anthropic 服务器上，本机查不到。
