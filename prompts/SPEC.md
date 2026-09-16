# 通用规格：会话查看器（clses 式）

所有目标工具共用这套功能规格。每份 `build-<工具>.md` 只改"数据源 / 恢复命令 / 活跃判定"三处，其余照此实现。

## 产物
- 单文件 Python 脚本，**纯标准库**，Python 3.8+，无第三方依赖
- 装到 `~/.local/bin/`（Windows: `%USERPROFILE%\.local\bin\` + 一个 `.cmd` 壳）
- 不写死任何绝对路径：一律 `os.path.expanduser`

## 命令
```
<cmd>                  列当前项目(目录)的会话，按最近活动排序
<cmd> N                恢复第 N 个会话(切到其原 cwd 后执行该工具的 resume)
<cmd> -x N, --view N   展开第 N 个会话的全文(user/assistant 文本 + 工具调用标记)
<cmd> -d N, --delete N 删除第 N 个会话(进回收站, 不直删; -y 跳过确认)
<cmd> --restore [N]    从回收站恢复; 省略 N 则列出回收站
<cmd> --trash          列出回收站
<cmd> -a, --all        列所有项目的会话(多一列"项目")
<cmd> -p PATH          列指定目录的会话
<cmd> -f, --full       加宽文本列
<cmd> -g PAT           按关键词过滤(主题/最后对话; 有 FTS 表的工具用全文匹配)
<cmd> -t RANGE         按时间过滤: today / Nh / Nd / Nw
<cmd> --sort KEY      time(默认) / oldest / size / turns
<cmd> --json / --csv   机器可读输出
<cmd> -y, --yes        删除跳过确认
<cmd> -h, --help       帮助
```
- `-v` 留空(习惯是 --version), 全文查看用 `-x`
- 动作(恢复/查看/删除/restore/trash)与列表选项(-a/-p/-f/-g/-t/--sort/--json/--csv)互斥

## 表格列
`# | Session ID | [项目] | 主题 | 最后对话 | 轮次 | 最后活动`
- **主题** = 首条真实用户提示
- **最后对话** = 最后一条用户提示
- **轮次** = assistant 消息条数
- **项目** 列仅在 `-a` 出现, 取会话原 cwd 的 basename
- 正在运行的会话在 `#` 列加 `*`

## 渲染
- **CJK 宽度感知**: `unicodedata.east_asian_width`, W/F 算 2 格
- **动态列宽**: `shutil.get_terminal_size`, 固定列+边框开销扣掉后, 主题≈1/3、最后对话≈2/3, clamp 到 [min,max]; `-f` 提高上限; 管道回退 120 列
- **颜色仅 tty**: 表头粗体、活跃红、ID/轮次/时间灰、末对话青; pad 之后再上色, 不破坏对齐

## 删除 = 回收站, 不是 rm
- 移到 `~/.<工具>-trash/<epoch>_<id>/` + meta.json, 可 `--restore` 移回
- 删前**活跃校验**: 正在运行的会话拒绝删除
- 目标必须在数据源根目录之下才允许删(防误删)
- 交互确认默认 N; **Ctrl+C = 取消**(不抛 traceback)

## 恢复
`os.chdir(该会话原 cwd)` 后用 `os.execvp(<工具resume命令>)` 直接替换进程(Linux/Mac)。Windows 用 `subprocess.call` + `shutil.which` 取真实 .exe 路径(见 build-claudecode.md 的 Windows 段)。

## 缓存
`~/.<cmd>-cache.json` 存序号→id + id→cwd + 真实数据路径 + 上次列表参数, 供 `N`/`-d`/`-x` 解析。序号对应"刚看到的那张表"。

## 5 个跨平台/数据源坑(必须处理)
1. **cwd→目录编码规则**(针对按目录分文件的存储, 如 Claude Code): 所有非字母数字字符→`-`, 不只是 `/`(`.`、`_` 也换)。再缓存真实路径双保险。
2. **活跃判定**因工具而异(锁文件 / sessions 表 / PID 存活), 见各 build 文件。
3. **删除时的文件锁**(Windows + 按文件存储): 进程正开着句柄时 move 会失败, 活跃校验要先过。
4. **颜色 ANSI**(Windows cmd): 启动加 `os.system("")` 启用 VT。
5. **不要用 `os.kill`**(Windows 没有): 用 `ctypes.windll.kernel32.OpenProcess + GetExitCodeProcess == 259`。
