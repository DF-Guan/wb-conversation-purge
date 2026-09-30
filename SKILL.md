---
name: wb-conversation-purge
display_name: WorkBuddy 对话记录彻底清理
display_name_en: WorkBuddy Conversation Purge
description: "彻底删除 WorkBuddy 本地对话记录与缓存。界面里的「删除」只是软删除（sessions.deleted_at 打标记），对话正文 .jsonl 仍以明文留在硬盘；本技能用于真正从硬盘清除对话、回收磁盘空间、抹去隐私残留。支持只清缓存、按时间清理（30d/6m/1y）、按体积筛选 Top N、清理界面已删除项，删除前自动备份、支持一键还原。当用户说「清理对话」「删除聊天记录」「彻底删除会话」「隐私清理」「清理缓存」「释放磁盘空间」「对话还在硬盘上」「conversation cleanup」「purge chat history」「clear WorkBuddy cache」时使用本技能。"
description_zh: "一键彻底删除 WorkBuddy 本地对话与缓存：界面删除只是软删除，对话正文仍以明文留在硬盘。先预览清单、经确认后自动备份再删除，支持按时间/体积/软删除筛选，可一键还原，安全回收磁盘空间、抹去隐私残留。"
description_en: "Truly purge WorkBuddy local chat history and caches from disk. The UI delete is only a soft-delete flag; plaintext .jsonl transcripts remain on disk. This skill previews a plan, backs up, then purges selected conversations or caches, with time/size filters and one-click restore."
when_to_use: "清理对话, 删除聊天记录, 彻底删除, 释放空间, 隐私清理, 对话还在硬盘上, 清理缓存, conversation cleanup, purge chat history"
examples_zh: ["帮我彻底清理 WorkBuddy 的对话记录", "清理一下 WorkBuddy 缓存，磁盘快满了", "把 30 天前的对话从硬盘上删掉", "看看哪些对话最占空间"]
examples_en: ["Purge my WorkBuddy chat history from disk", "Free up disk space by clearing WorkBuddy caches", "Delete conversations older than 30 days for real"]
category: productivity-tools
version: 2.1.0
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
| 附件缓存（增长最快） | `~/.workbuddy/blobs`、`file-history`、`traces`、`artifact-index` |

## 第零步：确定脚本绝对路径（不要跳过）

调用技能不会切换工作目录，`scripts/cleanup.py` 这类相对路径会直接找不到文件。加载本技能时，宿主会把 `${CLAUDE_SKILL_DIR}` 插值为本 SKILL.md 所在目录（WorkBuddy 与 Claude Code 均支持）；若宿主未插值，则手动拼出技能根目录绝对路径代替。后续所有命令都用这个绝对路径，**不要先 `cd` 到别处再指望相对路径生效**。

## 标准流程（五步，必须按序）

1. **预览（永远不要跳过）**
   ```bash
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --list
   ```
   列出对话与缓存各自占用多少。**把清单展示给用户，协助确认清理范围。** 想看谁最占地方，加 `--top 10`（按体积倒序取前 N）。

2. **确认范围** —— 与用户敲定用哪一档：只清缓存 / 只清界面已删除项 / 按时间 / 全部 / 是否连界面列表一起清。未经用户明确同意，不得进入下一步。

3. **提醒完全退出 WorkBuddy**（含托盘图标）。脚本检测到进程在跑会自行中止。

4. **备份后删除** —— 默认带 `--backup`（除非用户明确说不备份），加 `--yes` 跳过脚本内的二次输入：
   ```bash
   # 只清缓存，保留所有对话（最常用，缓存增长最快）
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --cache-only --backup --yes

   # 只清理界面里已删除的对话
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --soft-deleted --backup --yes

   # 清理早于指定时间的对话（支持 30d / 6m / 1y）
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --older-than 30d --backup --yes

   # 清理全部对话；加 --include-cache 连缓存一起；加 --purge-db 连界面列表一起清
   python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --all --include-cache --backup --yes
   ```

5. **核验并告知** —— 重新 `--list` 复查已清空；向用户报告：释放了多少空间、备份位置（`~/Desktop/WB-Cleanup-Backup/<时间戳>/`）、审计日志（`~/.workbuddy/cleanup-audit.log`）、以及如何用 `--restore` 还原。

## 还原

```bash
python "${CLAUDE_SKILL_DIR}/scripts/cleanup.py" --restore "<备份目录>"
```

备份目录里必须有 `_manifest.txt`（记录原始路径），脚本据此把文件放回原位。

## 参数

| 参数 | 作用 |
|---|---|
| `--list` | 只预览，绝不删除。可与其他模式组合（如 `--list --cache-only`） |
| `--cache-only` | 只清缓存目录，保留所有对话 |
| `--soft-deleted` | 只处理 `deleted_at` 非空的会话 |
| `--older-than AGE` | 只处理早于该时间的会话，支持 `30d` / `6m` / `1y` |
| `--all` | 处理所有会话 |
| `--top N` | 按体积倒序只取前 N 项 |
| `--include-cache` | 附带清理 blobs / file-history / traces / artifact-index |
| `--backup` | 删除前备份到 `~/Desktop/WB-Cleanup-Backup/<时间戳>/`（含 `_manifest.txt`）。缓存目录为可再生数据，整体删除、不做文件级备份 |
| `--restore DIR` | 从备份目录还原（依赖其中的 `_manifest.txt`） |
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
| 相对路径报「找不到脚本」 | 调用技能不切换 cwd，必须用绝对路径，见第零步 |
| 报删除失败 | 多半实际已删除。删完必须重新 `--list` 复查，别只看报错 |
| 界面列表还在 | 正常。清文件不同步删列表；想连列表一起清加 `--purge-db` |
| 「莫名没删掉」 | WorkBuddy 没退干净，文件被占用 |
| 当前会话的文件删不掉 | 正在使用中，退出后即可删，不必单独处理 |
| 清单里出现陌生文件夹名 | 该会话在数据库里没有标题，退回显示项目文件夹名 |

## 顺序铁律

**先在界面里删（打软删除标记），再跑脚本删文件。** 反过来会导致界面列表和磁盘状态对不上。
