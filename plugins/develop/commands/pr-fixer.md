---
name: pr-fixer
description: 读取远端仓库 PR 的 review comments，分两阶段整理并应用代码修改。当用户提供 PR 号码并要求根据 PR 评论修改代码时使用。关键词：PR、review、代码评审、comment、修改。
disable-model-invocation: true
---

## 概述

分两阶段处理 PR 评论：
1. **分析阶段**：读取所有 review comments，整理并说明需要修改的内容，等待用户确认
2. **修改阶段**：用户确认后，根据评论内容修改代码

使用 `gh` CLI 读取 PR 评论，不需要额外工具。

---

## 前置条件

- 已安装并登录 `gh` CLI
- 当前目录位于对应的 git 仓库中

```bash
gh auth status
```

---

## 第零步：确认当前分支

**在读取评论之前**，先确认当前分支是否为 PR 的来源分支：

```bash
CURRENT=$(git branch --show-current)
gh pr view <PR_NUMBER> --json headRefName -q .headRefName
```

若不一致，告知用户并询问是否切换分支后再继续。

---

## 第零步 补充：读取相关 spec 文件

从 PR 标题和分支名推断功能关键词，在 `docs/spec/` 下搜索相关规格文件：

```bash
# 确认分支名和 PR 标题
git branch --show-current
gh pr view <PR_NUMBER> --json title -q .title

# 搜索 docs/spec/ 下的 .md 文件
find ./docs/spec -type f -name '*.md' 2>/dev/null
```

- **找到相关 spec** → 读取内容，用于判断评论是否与规格一致（评论与 spec 矛盾时，在分析表格中注明）
- **未找到** → 跳过，仅依据代码上下文判断

---

## 第一阶段：分析 PR 评论

### 步骤 1：获取未解决的 review 评论

使用 GraphQL 只获取 **未解决（isResolved: false）** 的 review thread，以及所有一般评论：

```bash
PR=<PR_NUMBER>
OWNER=$(gh repo view --json owner -q .owner.login)
REPO_NAME=$(gh repo view --json name -q .name)

echo "=== PR 基本信息 ==="
gh pr view $PR --json title,state,baseRefName,headRefName

echo "=== 未解決の Review スレッド ==="
gh api graphql -f query="
{
  repository(owner: \"$OWNER\", name: \"$REPO_NAME\") {
    pullRequest(number: $PR) {
      reviewThreads(first: 50) {
        nodes {
          isResolved
          comments(first: 5) {
            nodes {
              body
              path
              line
              author { login }
            }
          }
        }
      }
    }
  }
}" --jq '
  .data.repository.pullRequest.reviewThreads.nodes[]
  | select(.isResolved == false)
  | .comments.nodes[0]
  | "文件: \(.path)\n行号: \(.line // "N/A")\n评论者: \(.author.login)\n内容: \(.body)\n---"
'

echo "=== 一般评论 ==="
gh pr view $PR --json comments \
  --jq '.comments[] | "评论者: \(.author.login)\n内容: \(.body)\n---"'
```

### 步骤 2：分析评论状态

收集到所有未解决评论后，综合两个维度判断处理方式：

**判断维度：**
1. **GitHub 解决状态**：`isResolved: false` 的才进入此步骤（已在步骤 1 过滤）
2. **代码实际情况**：读取评论对应的代码文件，判断问题是否在代码层面已被处理

最终状态：
- **需要修改**：代码未处理该评论的问题
- **已修改（代码已处理）**：读取代码后发现问题已被修复，虽 GitHub 未标 resolved，可建议用户手动 resolve
- **无需修改**：属建议性质、已过时、或代码逻辑本来就正确

输出格式：

```
## PR #<PR_NUMBER> 评论分析

### 需要修改的项目

| # | 文件 | 行号 | 评论者 | 评论内容 | 建议修改方式 |
|---|------|------|--------|----------|------------|
| 1 | path/to/file.py | 42 | reviewer | "变量名不清晰" | 将 `x` 改为 `user_count` |

### 已修改 / 无需修改的项目

| # | 文件 | 行号 | 评论内容 | 状态 | 判断理由 |
|---|------|------|----------|------|----------|
| 2 | path/to/file.py | 88 | "缺少空值检查" | ✅ 已修改 | 第 90 行已有检查 |
| 3 | path/to/file.py | 15 | "可以用列表推导式" | ⏭️ 无需修改 | 属建议性评论 |

---
以上是对 PR 评论的整理。请确认是否要根据以上内容进行代码修改？
```

**重要：展示分析结果后必须停下来等待用户确认，不要立即开始修改代码。**

---

## 第二阶段：应用代码修改

**只有在用户明确确认后才执行此阶段。**

### 修改原则

- 按文件分组处理，先读取文件确认上下文
- 只改评论指出的问题，不做额外重构
- 评论含糊时，根据代码上下文做最合理的判断

### 完成后汇报格式

```
## 代码修改完成

### 修改清单
- [x] `path/to/file1.py` - 行 42：将 `x` 改为 `user_count`
- [x] `path/to/file2.py` - 行 15：添加空值检查

### 未处理项目
（无法自动处理的评论在此说明原因）

---

### 建议的 Commit 分组

**Commit 1**（命名規範の修正）
fix: 変数名をより明確な命名に修正
- `path/to/file1.py`

**Commit 2**（バリデーション追加）
fix: 空値チェックを追加してNoneエラーを防止
- `path/to/file2.py`
```

### 测试执行（必须）

**修改完成后，必须执行测试并确认全件通过：**

```bash
task test
# または直接実行
cd src && uv run pytest
```

- **全件通过** → 报告 `✅ テスト全件パス（<N> passed）`
- **失败** → 报告失败内容并询问用户是否回滚该修改

---

## 第三步：记录 Reviewer 风格偏好

修改完成后，从评论中萃取 reviewer 的代码风格偏好，追加写入：

```
~/.claude/style-prefs/<repo-name>.md
```

萃取规则与档案格式详见 [REFERENCE.md](REFERENCE.md)。

---

## 使用流程

```
用户: /me:pr-fixer 123
  ↓
Agent: 确认当前分支 = PR 来源分支
  ↓
Agent: 读取 PR #123 的所有评论
  ↓
Agent: 整理分析表格 → 【停止，等待用户确认】
  ↓
用户: 确认（或提出调整意见）
  ↓
Agent: 逐一修改代码文件
  ↓
Agent: 执行测试，确认全件通过
  ↓
Agent: 汇报修改结果 + 建议 commit 分组
  ↓
Agent: 萃取风格偏好 → 追加写入 ~/.claude/style-prefs/<repo>.md
  ↓
Agent: 建議用戶使用 /me:style-check 做程式碼風格檢查
```

---

## 注意事项

- **必须两阶段执行**：先分析展示，等用户确认后再修改
- 若 PR 评论中有矛盾或不明确的地方，在分析阶段就指出，请用户说明优先级
- 若评论要求的修改影响范围较大，在分析阶段提醒用户
