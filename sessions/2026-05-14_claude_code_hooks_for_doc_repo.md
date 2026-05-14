# 2026-05-14 深夜 Claude Code hook 配置：start-work 前自动 pull / end-work 后自动 push

接续当晚 xlsx 补列工作 + end-work 归档后的工作流改进。本次目标：让 doc 仓库的拉取/推送自动化，免去手动操作。改动落在 `snowmeet_ai/.claude/settings.local.json`（本机个人配置，gitignore）。

## 1. start-work 前自动 pull

### 1.1 需求

用户："每次执行 start-work 之前，自动在 snowmeet_ai_doc 目录下执行 git pull"

意图：避免在过期 CLAUDE.md 上继续工作，导致与远端冲突。

### 1.2 实现

`PreToolUse` hook，matcher `Skill`，`if: "Skill(start-work)"` 过滤：

```json
{
  "hooks": {
    "PreToolUse": [{
      "matcher": "Skill",
      "hooks": [{
        "type": "command",
        "command": "git -C /Users/cangjie/source/snowmeet/snowmeet_ai/snowmeet_ai_doc pull --ff-only 2>&1 || echo '[warn] git pull failed (网络/冲突)，继续 start-work'",
        "if": "Skill(start-work)",
        "statusMessage": "拉取 snowmeet_ai_doc 最新提交..."
      }]
    }]
  }
}
```

- `--ff-only`：拒绝产生 merge commit。本地有未推送提交或分叉就 abort，让用户决定 merge / rebase 策略
- `2>&1 || echo ...`：失败（网络断 / fast-forward 拒绝）打 warn 不阻断 start-work
- `if` 字段是 permission rule 语法，可以精确匹配 `Skill(start-work)`，避免对其它 skill 误触

### 1.3 验证

- pipe-test：`echo '{"tool_name":"Skill","tool_input":{"skill":"start-work"}}' | <command>` → `Already up to date.` exit=0
- `jq -e '.hooks.PreToolUse[]...' settings.local.json` 校验 schema → exit=0

## 2. end-work 后自动 push（关键设计决策）

### 2.1 需求

用户："每次执行 end-work 之后，自动在 snowmeet_ai_doc 这个目录下自动执行 git push"

后续补充："push 之前，要看看还有什么需要添加的新增的文件，如果有，执行 git add ."

### 2.2 时机难题（PostToolUse + Skill 不行）

第一反应：和 start-work 对称写 `PostToolUse + Skill(end-work)`。

**但这不行**：Skill 工具的 PostToolUse 在 skill **工具返回**那一刻触发 —— 此时只是 skill 指令文本被加载进 Claude context，Claude **尚未执行**写 CLAUDE.md / 写 sessions 文件等步骤。git push 在那个时点什么都没得推。

### 2.3 改用 Stop hook + 启发式

`Stop` hook 在每次 Claude 停下时触发（end-work 写完文件 + 总结回复后会停）。问题是 Stop 也包含普通对话停下、/clear、/compact 等场景，不能无脑 push。

启发式信号：**`snowmeet_ai_doc/sessions/*.md` 在最近 3 分钟有修改** = end-work 刚跑过。

```bash
DOC=/Users/cangjie/source/snowmeet/snowmeet_ai/snowmeet_ai_doc
if find "$DOC/sessions" -maxdepth 1 -name '*.md' -mmin -3 2>/dev/null | grep -q .; then
  status=$(git -C $DOC status --porcelain)
  if [ -n "$status" ]; then
    echo "[hook] 检测到改动 (?? = 新文件, M = 修改, D = 删除):"
    echo "$status"
    git -C $DOC add . && git -C $DOC commit -m "auto: end-work session archive $(date +%Y-%m-%d_%H%M)" 2>&1
  fi
  git -C $DOC push 2>&1 || echo '[warn] git push failed'
fi
```

- 外层 `find -mmin -3`：只在 end-work 信号出现时进入分支，普通停下不会误触
- 内层 `status --porcelain`：有改动才 commit；干净就只 push（cover 上次 commit 成功但 push 失败的边缘场景）
- `git add .` 用显式形式（用户偏好），等价于 `git -C $REPO add -A`
- 输出含改动列表，可视化 hook 行为

### 2.4 验证

- pipe-test：当前 sessions/ 最近修改文件 mtime 已超 3 分钟窗口 → 跳过分支 → exit=0，符合预期
- `jq -e` 校验 schema → exit=0

## 3. 副产物：发现 doc 仓库残留 merge conflict

本次 end-work 改 CLAUDE.md 时 Edit 报 "File has been modified since read"，重读发现 lines 174-178 有：

```
<<<<<<< HEAD
## 当前状态（截至 2026-05-14 晚）
=======
## 当前状态（截至 2026-05-14 晚上）
>>>>>>> 50ba79589195643a8395f648b59ac887ed9fe011
```

`git log` 显示 `d0d80e1 Merge branch 'main' of github.com:cangjie/snowmeet_ai_doc` —— 这个 merge commit 是在残留冲突标记**未解决**就被 commit 进去的。`git status` 显示干净，所以工具看不出问题。

**修复**：手动 Edit 把 5 行冲突标记替换为本次 end-work 的新标题 `## 当前状态（截至 2026-05-14 深夜）`，统一三方分歧。

**教训**：merge 后必须 `git status` 看 unmerged paths、grep 冲突标记，不能直接 commit。后续如发现类似情况优先怀疑此根因。

## 关键改动文件

| 文件 | 改动 |
|---|---|
| `.claude/settings.local.json` | 新增 PreToolUse + Stop hook（在已有 permissions 块基础上 merge） |
| `snowmeet_ai_doc/CLAUDE.md` | 解决 lines 174-178 merge conflict；当前状态戳 → 深夜；追加本次开发日志条目 |

## 学到的小知识

1. **Skill 工具的 PostToolUse 时机不等于 skill 工作流完成**：Skill tool 返回 = 指令文本加载进 context，模型还没执行任何 skill 步骤。"在 skill 之后做 X" 的需求要绕开这个时机：Stop hook + 启发式信号、或 Write/Edit hook + 路径过滤
2. **Stop hook 的幂等性设计**：Stop 高频触发（每次模型停下），不能无脑做副作用动作。要么外层加 specific signal 过滤（如本次的 sessions/ mtime），要么动作本身天然 no-op（如 push working tree clean 时输出 "Everything up-to-date"）
3. **hook 命令的 if 字段用 permission rule 语法**：`"if": "Skill(start-work)"` / `"if": "Bash(git *)"` 精确匹配，避免对全部同 matcher 的 tool call 都触发。比在 command 内部 grep tool_input 优雅
4. **`git add .` ≈ `git add -A`**（在 `git -C $REPO` 下）：两者在 repo root 下功能一致，都 stage 新增/修改/删除。新文件（untracked）由 `git status --porcelain` 的 `??` 状态码呈现
5. **`--ff-only` 是自动 pull 的安全护栏**：本地有未推送提交时拒绝合并而不是产 merge commit，强制人工干预决定 rebase / merge 策略
6. **`.claude/settings.local.json` 是本机个人配置**：gitignored，不入库，团队不共享。适合放自动化 hook、本机权限白名单、本机环境变量
