---
name: wb-conversation-purge
display_name: WorkBuddy 对话记录彻底清理
display_name_en: WorkBuddy Conversation Purge
description: "清理 WorkBuddy 缓存与对话记录、释放磁盘空间（真删除，非界面软删除）。用户只要提到「清理 WorkBuddy 缓存」「清理对话」「删除聊天记录」「释放空间」「清理日志/图片缓存」「对话还在硬盘上」，就优先使用本技能，不要改用 PowerShell/rm 手工扫描清理——本技能已内置全部可清理项识别、占用文件自动跳过、备份、审计与还原，一条命令完成。覆盖：软删除残留对话、blobs 图片缓存、traces、artifact-index、cache、shell-snapshots、轮转日志；默认不碰回滚数据与程序本体。"
description_zh: "一键清理 WorkBuddy 缓存与对话记录：自动识别全部可再生缓存（图片、追踪、轮转日志等）与界面已删残留，运行中即可执行、占用文件自动跳过，附带备份、审计与还原。默认不触碰回滚数据和程序本体。"
description_en: "One-command cleanup of WorkBuddy caches and chat history: identifies every regenerable cache (images, traces, rotated logs) plus soft-deleted conversations, works while the app is running, skips locked files automatically, and ships with backup, audit log and one-click restore."
when_to_use: "清理对话, 删除聊天记录, 彻底删除, 释放空间, 隐私清理, 对话还在硬盘上, 清理缓存, conversation cleanup, purge chat history"
examples_zh: ["帮我彻底清理 WorkBuddy 的对话记录", "清理一下 WorkBuddy 缓存，磁盘快满了", "把 30 天前的对话从硬盘上删掉"]
examples_en: ["Purge my WorkBuddy chat history from disk", "Free up disk space by clearing WorkBuddy caches", "Delete conversations older than 30 days for real"]
category: productivity-tools
version: 2.4.0
author: DF
allowed-tools: Bash(python:*)
agent_created: true
---

# WorkBuddy 对话记录彻底清理

界面上的「删除」**不会**删掉本地文件，只是给数据库打软删除标记。真正的数据在硬盘上：

| 内容 | 位置 |
|---|---|
| 对话正文（明文 JSONL，含工具调用完整出入参） | `~/.workbuddy/projects/<项目ID>/<会话UUID>.jsonl` |
| 回滚快照 | 同目录 `<UUID>.file-rollback.ndjson` |
| 会话元信息 | 同目录 `<UUID>.meta.json` |
| 软删除标记 | `~/.workbuddy/workbuddy.db` → `sessions.deleted_at` |
| 附件与缓存（增长最快） | `~/.workbuddy/blobs`、`traces`、`artifact-index`、`cache`、`shell-snapshots`、`logs/*.old.log` |

## 第零步：确定脚本绝对路径（不要跳过）

调用技能不会切换工作目录，`scripts/cleanup.py` 这类相对路径会直接找不到文件。加载本技能时，宿主会把 `${CLAUDE_SKILL_DIR}` 插值为本 SKILL.md 所在目录（WorkBuddy 与 Claude Code 均支持）；若宿主未插值，则手动拼出技能根目录绝对路径代替。后续所有命令都用这个绝对路径，**不要先 `cd` 到别处再指望相对路径生效**。

## 标准流程（五步，必须按序）

1. **预览（永远不要跳过）**
   ```bash
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --list
   ```
   列出对话与缓存各自占用多少。**把清单展示给用户，协助确认清理范围。** 想看谁最占地方，加 `--top 10`（按体积倒序取前 N）。

2. **确认范围** —— 与用户敲定用哪一档：只清缓存 / 只清界面已删除项 / 按时间 / 全部 / 是否连界面列表一起清。未经用户明确同意，不得进入下一步。

3. **直接执行** —— WorkBuddy 运行中也能清理：被占用的文件（如当前对话）会自动跳过并标注 `[跳过]`，其余照常删除。仅当使用 `--purge-db`、或要求 100% 清理干净时，才需要先完全退出 WorkBuddy（含托盘图标）。

4. **备份后删除** —— 默认带 `--backup`（除非用户明确说不备份），加 `--yes` 跳过脚本内的二次输入：
   ```bash
   # 【首选·一键】界面已删对话 + 全部可再生缓存（含轮转日志）
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --auto --backup --yes

   # 只清缓存，保留所有对话（场景：只想释放空间）
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --cache-only --backup --yes

   # 只清理界面里已删除的对话
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --soft-deleted --backup --yes

   # 清理早于指定时间的对话（支持 30d / 6m / 1y）
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --older-than 30d --backup --yes

   # 清理全部对话；加 --purge-db 连界面列表一起清
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --all --backup --yes
   ```

5. **核验并告知** —— 重新 `--list` 复查；向用户报告脚本给出的结果：**释放量（对话/缓存分账）、根目录清理前后对比**、备份位置（`~/Desktop/WB-Cleanup-Backup/<时间戳>/`）、审计日志（`~/.workbuddy/cleanup-audit.log`）、以及如何用 `--restore` / `--restore-latest` 还原。

## 还原

```bash
# 不知道备份时间戳？先列出所有备份，再还原最近一次
python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --list-backups
python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --restore-latest

# 或用指定目录还原
python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --restore "<备份目录>"
```

备份目录里必须有 `_manifest.txt`（记录原始路径），脚本据此把文件放回原位。

## 参数

| 参数 | 作用 |
|---|---|
| `--list` | 只预览，绝不删除。可与其他模式组合（如 `--list --cache-only`） |
| `--cache-only` | 只清缓存目录，保留所有对话 |
| `--soft-deleted` | 只处理 `deleted_at` 非空的会话 |
| `--auto` | **一键**：界面已删对话 + 全部可再生缓存（含轮转日志），最省事的默认档 |
| `--older-than AGE` | 只处理早于该时间的会话，支持 `30d` / `6m` / `1y` |
| `--all` | 处理所有会话 |
| `--top N` | 按体积倒序只取前 N 项 |
| `--include-cache` | 附带清理 blobs / traces / artifact-index / cache / shell-snapshots / 轮转日志（`file-history`、`changes-detail` 属回滚数据，默认不动） |
| `--backup` | 删除前备份到 `~/Desktop/WB-Cleanup-Backup/<时间戳>/`（含 `_manifest.txt`）。缓存目录为可再生数据，整体删除、不做文件级备份 |
| `--restore DIR` | 从备份目录还原（依赖其中的 `_manifest.txt`） |
| `--restore-latest` | **还原最近一次备份**（不用记时间戳目录） |
| `--list-backups` | 列出所有备份目录，标注哪些可还原 |
| `--purge-db` | 额外清空数据库里的会话列表（会先备份数据库） |
| `--yes` | 跳过 DELETE 确认。仅当用户已在对话中确认后使用 |
| `--backup-dir PATH` | 自定义备份根目录 |

## 硬规则（红线）

- **绝不跳过预览直接删。** 没有把清单展示给用户，就不允许执行删除。
- **`--yes` 不等于授权**，它只省掉脚本内的二次输入；真正的授权来自用户在对话里的明确同意。
- **进程占用时必须中止**，让用户退出 WorkBuddy；不要强删被占用的文件。
- **默认带 `--backup`**，除非用户明确说不需要备份。
- **不碰 `~/.workbuddy` 之外的数据**，不删 `skills/`、`connectors/`、`credentials/`，不改其他系统文件。

## 常见坑

| 现象 | 处理 |
|---|---|
| **想手工清** | **别用 PowerShell/rm 直接扫 `~/.workbuddy`**：容易误删程序本体（`workspace`/`binaries`/`plugins`/`app`），且无备份、无审计。一律用本技能，一条命令覆盖全部可清理项 |
| 相对路径报「找不到脚本」 | 调用技能不切换 cwd，必须用绝对路径，见第零步 |
| 报删除失败 | 多半实际已删除。删完必须重新 `--list` 复查，别只看报错 |
| 界面列表还在 | 正常。清文件不同步删列表；想连列表一起清加 `--purge-db` |
| 「清理后有残留」 | 运行中被占用的文件会标 `[跳过] 被占用`；想补清就退出 WorkBuddy 重跑同一命令 |
| 当前会话的文件删不掉 | 正在使用中，运行中自动跳过；退出后即可删，不必单独处理 |
| 清单里出现陌生文件夹名 | 该会话在数据库里没有标题，退回显示项目文件夹名 |

## 顺序铁律

**先在界面里删（打软删除标记），再跑脚本删文件。** 反过来会导致界面列表和磁盘状态对不上。
