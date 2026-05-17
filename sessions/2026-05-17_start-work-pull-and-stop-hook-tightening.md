# 2026-05-17 start-work 内置 git pull + Stop hook 收紧：从「加载到过期上下文」追到自动 push 工作流并修正

本场会话从一次 `start-work` 说起：它加载了过期的 CLAUDE.md。顺着「为什么没自动 pull」一路排查，定位到同步本应由 `.claude/settings.local.json` 的 hook 完成但未触发，按用户要求做了两处修正：① 把 `git pull` 写进 start-work 的 SKILL.md 第 1 步；② 收紧 Stop hook，使其只归档 `sessions/`+`CLAUDE.md`，不再 `git add .` 吞掉无关 WIP 并自动 push。改动落在 `snowmeet_ai_doc/.claude/skills/start-work/SKILL.md`（入库）与仓库根 `.claude/settings.local.json`（gitignored/机器本地）。

## 1. 触发点：start-work 读到旧上下文

- 执行 start-work，先选了「财年导出收尾」方向并做了调研（FY 脚本是唯一带 openid 列的，两级偏好口径内联在 `export_rent_orders_fy.py:187-201`；通用导出脚本无 openid 列），但用户中断：「最近的工作结果没有同步下来，一会儿我会再执行下 start-work」。
- 用户追问核心问题：**「目前的 start-work 第一步为什么不是 git pull？」**

## 2. 排查：为什么没 pull / 没同步

### 2.1 设计上 pull 是 hook 不是 skill 步骤

- `SKILL.md` 体内只「读 + 展示 CLAUDE.md」，无副作用。
- auto-pull 设计为 `PreToolUse / Skill(start-work)` hook（`.claude/settings.local.json`），由 harness 在 skill 指令进 context 前确定性执行。理由见 CLAUDE.md 2026-05-14（深夜）：hook 不可跳过，skill 步骤只是模型「可能照做」。

### 2.2 实测这次根本没同步

- `git status` 报 `0 0 / up to date`，但那是**本地缓存的 origin/main**。
- `git ls-remote origin refs/heads/main`（只读）查真实远端：
  - 会话起始本地 HEAD=`ffbb27e`，缓存 origin/main=`ffbb27e`，**真实远端=`dbaa546`** → 连 fetch 都没发生。
- 结论：本会话 PreToolUse hook **未执行**。最可疑：该 hook entry 用了非标准的 `"if": "Skill(start-work)"` 键 + `|| echo warn` 吞错不阻断（任何失败都静默继续）。

## 3. 修正 1：git pull 写进 start-work SKILL.md 第 1 步（用户指定）

- 用户明确：「你就把 git pull 写在你的 start-work 这个 skill 的第一句即可。」
- 改 `snowmeet_ai_doc/.claude/skills/start-work/SKILL.md` 的 `## Process`：
  - 新增第 1 步 `git -C snowmeet_ai_doc pull --ff-only`（从仓库根定位，不用历史遗留 Mac 绝对路径）；`--ff-only` 防分叉乱产 merge commit；失败时**显式告警**「⚠️ 同步失败，下方上下文可能过期」再继续，不静默吞错。
  - 原 Read / Present / Format 顺延为第 2 / 3 / 4 步。
- 理由：skill 入库、跨机生效；不再依赖 gitignored/机器本地、且本会话实测不可靠的 hook。

## 4. 「我没看到你做任何改动」——再排查

- 用户反馈 `git status` 干净、看不到 SKILL.md 改动。
- 逐层查证：文件磁盘内容确有改动且被 git 跟踪、未被 gitignore；但 `git diff HEAD` 为空。
- 真相：HEAD 已是 **`e899295`**（msg `auto: end-work session archive 2026-05-17_2130`），其内容**逐字包含**我加的 Pull 步骤。
- 提交链线性无分叉：`e899295 → dbaa546 → ffbb27e`，`dbaa546` 是 HEAD 祖先，真实远端=本地 HEAD=`e899295`。
- 即：`.claude/settings.local.json` 的 **Stop hook**（sessions/*.md 近 3 分钟改动 → `git add . && commit && push`）在某回合结束时先吃进远端 `dbaa546`，再把我的 SKILL.md 改动作为 `e899295` 提交并 push 到共享远端。改动没丢、已同步——`git status` 干净正是因为 hook 已把它扫走。

## 5. 修正 2：收紧 Stop hook（用户要求「需要收紧」）

- 问题：Stop hook 用 `git add .`，会把任何 WIP（含有意的 skill 代码改动）以 `auto: end-work session archive` 名义自动推到共享远端，无确认。
- 经 update-config skill 改 `.claude/settings.local.json` 的 Stop hook command（结构/statusMessage 不变）：

  | 维度 | 旧 | 新 |
  |---|---|---|
  | 触发 | sessions/*.md 近 3 分钟改动 | 不变 |
  | 暂存 | `git add .` | `git add -- sessions CLAUDE.md` |
  | 提交 | 工作区非空就 commit | 仅这两路径有改动才 commit |
  | 推送 | 无条件 push | 仅 commit 成功后才 push |
  | 其余改动 | 一并 commit+push | `git status --porcelain` 列出，提示「留待手动处理」 |

- 验证：整文件合法 JSON（`jq -e`）、schema 路径可取出 command、`bash -n` 语法通过；PreToolUse(pull) hook 未被波及。
- 隔离临时仓库（含 fake origin）实跑两场景：
  - **A**（sessions+CLAUDE.md+无关 skill.md 同改）：提交只含 `CLAUDE.md`+`sessions/s1.md`，`skill.md` 留工作区并被列「留待手动」，push 成功 `72aeb79..ffd668e main -> main`。
  - **B**（仅无关 skill.md 改、有近期 session 文件）：无新提交、origin 未 push、skill.md 仅被提示。
  - （A 末尾测试脚本断言 `master`/`main` 分支名错位导致一行假阴性「NO 未同步」，git push 输出已确证实际成功，非 hook 缺陷。）

## 关键改动文件

| 文件 | 改动 |
|---|---|
| [`snowmeet_ai_doc/.claude/skills/start-work/SKILL.md`](.claude/skills/start-work/SKILL.md) | `## Process` 新增第 1 步 git pull（入库，已随 `e899295` 推送）|
| `/Users/cangjie/source/snowmeet/snowmeet_ai/.claude/settings.local.json` | Stop hook command 收紧为只 `git add -- sessions CLAUDE.md`（gitignored/机器本地，不入库）|

## 学到的小知识

1. **`git status` 的「up to date with origin/main」比的是本地缓存的 origin/main**：未 fetch 时会谎报同步。判断真实远端用 `git ls-remote origin refs/heads/main`（纯只读，不动 ref/工作树）。
2. **Stop hook 时机 ≠ end-work 完成**：旧设计用「sessions/ 近 3 分钟 mtime」启发式匹配真实写入；副作用是 `git add .` 会连带吞掉任何 WIP。收紧成路径白名单（sessions/+CLAUDE.md）后才安全。
3. **`.claude/settings.local.json` 是 gitignored / 机器本地 / 不跨机**：start-work 的 pull、end-work 的 push，可靠性必须落在「入库的 skill 步骤 + 跨会话记忆」，hook 仅作本机冗余。本会话 PreToolUse(pull) hook 实测未触发（疑非标准 `if` 键）。
4. **改动「看不到」未必是没改**：Stop hook 可能已 commit+push；先看 `git log`/`git diff HEAD` 与真实远端，再下结论。
