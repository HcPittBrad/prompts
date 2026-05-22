# issule-goal — 目标 → 可执行 Issue

**适用于任何类型任务。人只做两件事：提目标 + 确认标题。其余全是 agent 的事。**

遵循 [output-protocol.md](./output-protocol.md) LAW1 + LAW2。

---

## 调用方式

```
/issule-goal <idea>          # 首次调用：建北极星 Issue
/issule-goal <idea> <repo>   # 指定仓库
/issule-goal <idea> dry      # 只输出草稿，不创建
```

后续步骤执行时，agent 直接引用 issue URL，按步骤执行，每步遵循本文件「步骤执行协议」。

---

## 首次调用 — 北极星 Issue

### Phase 1 — 理解目标

必须回答这一个问题：

> 这个任务完成后，有一件事从「不存在 / 不正确 / 不可用」变成了「存在 / 正确 / 可用」。这件事是什么？

写不出来 → 继续追问，直到写出来。

### Phase 2 — 调研上下文

根据任务类型自动选择：

```
代码任务   → 读本地代码库（find + grep + 读关键文件）
              + 涉及第三方库时联网查最新文档
产品/设计  → 读现有文档 / 截图 / 用户反馈
研究/调研  → 联网搜索，来源优先官方文档 + 近12个月
其他       → 从对话上下文推断
```

### Phase 3 — 定义三重验证退出条件（契约核心）

**这是人机协作的契约。定义一次，贯穿整个任务生命周期，verify 时直接执行。**

每条必须是「操作 → 可判断的预期结果」格式：

```
T1 数字化验证（agent 自动跑，出数字）
   命令/查询 → 期望值（精确数字或布尔）
   例：SELECT count(*) FROM scenes WHERE image_prompt IS NOT NULL → 191
   例：tsc --noEmit → 0 errors
   例：curl /api/health → status: 200

T2 AI 推理验证（agent 自判，给置信度）
   判断标准（具体，不模糊）→ 置信度阈值
   例：抽查10条生成内容，判断是否符合业务语义 → 置信度 ≥ 85%
   例：diff 改动范围是否在 scope 内 → 置信度 ≥ 90%

T3 人工验证（给人精确步骤，不是判断题）
   第1步：[精确操作，打开哪个URL/执行什么]
   第2步：[看什么 / 找什么]
   预期：[符合则通过，不符合则描述X发给agent]
```

**三条原则：**
- T1 能写的不写 T3，人工验证成本最高
- T3 只给步骤，不问人「感觉对不对」
- 置信度 < 60% 的 T2 自动升级为 T3

### Phase 4 — 锁标题（先锁，再写正文）

格式：`{emoji} {类型}: {业务描述（≤20字，零技术词汇）}`

```
emoji 选择：
🔴 主流程阻断   🟡 核心功能缺失   🟢 体验提升
🔵 维护/重构    🟣 探索/调研
```

输出：
```
🎯 标题提案：[emoji] [type]: [描述]
确认即写正文；要改直接说。
```

标题锁定后不得修改。

### Phase 5 — 生成 Issue

**Issue 结构（人看顶部，agent 看底部）：**

```markdown
## 目标

让什么变成真的：[一句话，极限精确]

问题：[现在什么不对或缺什么]

---

## 三重验证退出条件（人机契约）

**T1 数字化：**
- `[命令]` → 期望：`[值]`

**T2 AI推理：**
- [判断标准] → 置信度 ≥ [X]%

**T3 人工：**
- 第1步：[操作]
- 第2步：[看什么]
- 预期：[标准]

---

## 执行步骤

> 每步遵循 output-protocol.md LAW1：目标→行动→结果→比对→闭环

**Step 1：[描述]**
[具体内容，代码任务含文件路径+操作位置]

**Step 2：[描述]**
[具体内容]

---

## 边界

不做：[out of scope 项] — 原因：[为什么]
```

### Phase 6 — 创建 GitHub Issue

```bash
# Step 6.1 Milestone（今天日期，不存在则创建）
TODAY=$(date +%Y-%m-%d)
MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  --jq ".[] | select(.title==\"$TODAY\") | .number")
[ -z "$MILESTONE_NUM" ] && MILESTONE_NUM=$(gh api repos/$REPO/milestones \
  -X POST -f title="$TODAY" --jq ".number")

# Step 6.2 查现有标签，AI 从中匹配（不新建）
gh label list --repo $REPO --json name --jq '.[].name'
```

调用 `mcp__github__create_issue`：
- title：Phase 4 锁定标题，原文照搬
- milestone：Step 6.1 拿到的数字 ID
- labels：从 repo 现有标签匹配，不新建

**创建后输出：**
```
✅ Issue #N 已创建

给 agent 的执行入口（复制第一行作为会话标题）：
──────────────────────────────────────
#N [issue 标题]
https://github.com/[owner]/[repo]/issues/N
──────────────────────────────────────
```

---

## 步骤执行协议（后续调用）

**每个 Step 是一个独立的 output-protocol.md LAW1 闭环：**

```
Step N 目标
    │
    ▼
行动（读/写/调用/验证）
    │
    ▼
结果
    │
    ▼
和 Step 目标比对
    │
有偏差？
YES          NO
 │            │
调整行动     Step N ✅ → 进 Step N+1
 │
 ▼
重新执行（最多3次，超过则中断报告）
```

**中断条件（遇到立刻停，不得自行决策）：**
- 必须修改 out of scope 内容才能继续
- 涉及不可逆操作（删数据/强推/破坏性迁移）
- 三重验证条件之间存在矛盾

---

## 行为准则

| 做 | 不做 |
|----|------|
| Phase 3 先写三重验证条件，这是契约 | 跳过验证条件直接写步骤 |
| Phase 4 先锁标题，等确认再写正文 | 标题和正文同时扔出来 |
| 每步遵循 LAW1 闭环 | 一口气执行多步不验证 |
| T1 能写的不写 T3 | 惯性把所有验收推给人 |
| 中断时给明确原因和选项 | 静默跳过或自行决策 |
