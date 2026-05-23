# Agent Orchestrator 架构图集

> 全局 Agent 编排系统设计，基于 GitHub Issue 作为唯一契约，分治法 + 总分总结构。

---

## 1. Issue 类型体系

```mermaid
graph LR
    RI["🔵 research\n调研阶段\n探索性·可多个·乱没关系"]
    MI["🟡 master\n= PRD·唯一契约\n贯穿派单→执行→验收"]
    CI["🟢 child\n原子执行任务\n单一职责·高度可执行"]
    SO["🔷 solo\n独立问题\n不走全局流程·直接执行"]
    IV["⚫ invalid\n作废"]

    RI -->|"/prd 综合 → 人确认"| MI
    MI -->|"/batch-issue 拆分"| CI
```

| Label | 阶段 | 数量 | 特征 |
|-------|------|------|------|
| `research` | 调研 | M个 | 探索·混乱·可并行·可废弃 |
| `master` | PRD→验收 | 1个 | 权威·干净·贯穿始终 |
| `child` | 执行 | N个 | 原子·单一职责·可独立执行 |
| `solo` | 任意 | 1个 | 独立·不走 Orchestrator |
| `invalid` | 任意 | - | 作废标记 |

---

## 2. 全局 Orchestrator — C4 L1 系统上下文

```mermaid
graph TB
    DEV(["👤 Developer"])

    subgraph ORCH["Orchestrator System"]
        O["全局编排器\n调研 → PRD → 派单 → 执行 → 验收"]
    end

    GH["GitHub Issues\n唯一事实标准"]
    CUR["Cursor Agents\n代码执行"]
    WEB["Web / Docs / GitHub历史\n公开信息源"]
    RAG["私有知识库\nmarkdown → pgvector\n私有数据资产"]

    DEV -->|输入问题域| O
    DEV -->|🔔 × 4 人工确认点| O
    O -->|读写 Master + Child Issues| GH
    O -->|派发执行任务| CUR
    O -->|调研公开信息| WEB
    O -->|查询私有资产| RAG
```

---

## 3. 全局 Orchestrator — C4 L2 容器

```mermaid
graph TB
    DEV(["👤 Developer"])

    subgraph ORCH["Orchestrator"]
        RC["Research Container\n/research\n并行调研·多源\n→ research-issues"]
        PC["PRD Container\n/prd\n综合报告→PRD\n→ master-issue body"]
        IF["Issue Factory\n/batch-issue\n唯一批量出口\n→ child-issues × N"]
        EX["Execution Container\n/cursor\n并行执行\n直接 commit"]
        VF["Verification Container\n/issule-verify\n对照 master AC\n无 PR"]
    end

    GH[("GitHub Issues\nresearch + master + child")]
    RAG[("知识库\n阶段1: markdown\n阶段2: Supabase pgvector")]

    DEV --> RC
    RC -->|调研写入 comment| GH
    RC <-->|查询私有资产| RAG
    PC -->|PRD 更新 body| GH
    IF -->|创建 child-issues| GH
    EX -->|执行报告写回 comment| GH
    VF -->|PASS: close all\nFAIL: re-queue| GH
```

---

## 4. C4 L3 — Research Container

```mermaid
graph TB
    subgraph RC["Research Container"]
        CL["Source Classifier\n判断信息源类型"]
        BA["/browser\n公开网页爬取"]
        GA["/gh\n历史 issues/PR 分析"]
        UA["User Input Agent\n结构化用户输入"]
        RA["RAG Agent\n私有知识库查询"]
        RM["Report Merger\n合并 → research-issue comment"]
    end

    IN["问题域输入"] --> CL
    CL --> BA & GA & UA & RA
    BA & GA & UA & RA -->|并行产出| RM
```

---

## 5. C4 L3 — Issue Factory

```mermaid
graph TB
    subgraph IF["Issue Factory"]
        PP["PRD Parser\n解析需求边界 + AC"]
        DE["Decomposer\nPRD → 原子任务列表"]
        TV["T-Validator\n单一职责检查\nAC + DoD 完整性"]
        GW["GitHub Writer\nbatch create child-issues"]
    end

    PRD["master-issue body\n(PRD)"] --> PP --> DE --> TV --> GW
```

---

## 6. Master Issue 状态机

```mermaid
stateDiagram-v2
    [*] --> researching : /research → research-issues 创建
    researching --> prd : 🔔 Human 确认调研
    prd --> ready : 🔔 Human 审核 PRD
    ready --> executing : 🔔 Human 确认 → /batch-issue + /cursor
    executing --> verifying : Cursor 完成 → 自动触发 /issule-verify
    verifying --> done : PASS → close all
    verifying --> executing : 🔔 FAIL → Human 确认重跑
    done --> [*]
```

> 🔔 = 必须人工确认，记录在 master-issue comment

---

## 7. 人机交互流（4个确认点）

```mermaid
sequenceDiagram
    actor Human
    participant RI as research-issues
    participant MI as master-issue (PRD)
    participant CI as child-issues
    participant EX as Cursor Agents

    Human->>RI: /research "问题域描述"
    Note over RI: 并行调研·多源·写 comment

    RI-->>Human: 🔔 调研完成，确认进入 PRD？

    Human->>MI: 确认 → /prd #research-id
    Note over MI: 综合→PRD·更新 body·label: master

    MI-->>Human: 🔔 PRD 已生成，请审核确认派单

    Human->>CI: 确认 → /batch-issue #master-id
    Note over CI: 拆分原子 child-issues·label: child

    CI-->>Human: 🔔 已创建 N 个任务，确认执行？

    Human->>EX: 确认 → /cursor --label child
    Note over EX: 并行执行·直接 commit·报告写回

    EX->>MI: /issule-verify 自动触发

    alt PASS
        MI-->>Human: 🔔 验收通过 → close all
    else FAIL
        MI-->>Human: 🔔 X 个任务失败，是否重跑？
        Human->>EX: /cursor 重跑
    end
```

---

## 8. 验收层（无 PR）

```mermaid
graph LR
    VF["/issule-verify"] -->|读| AC["master-issue AC"]
    VF -->|读| RP["child-issue 执行报告"]
    VF -->|PASS| CL["close child-issues\nclose master-issue\n写验收 comment"]
    VF -->|FAIL| RQ["re-open child-issues\n标注失败原因\nre-queue"]

    NOTE["PR 保留场景：\n人工需要看 diff 时手动开\n不进自动化流程"]
```

---

## 9. 私有知识库演进策略

```mermaid
graph LR
    subgraph S1["阶段1（立即可用）"]
        MD["research/*.md 文件\ngrep 搜索\n零基础设施"]
    end

    subgraph S2["阶段2（积累100+文档后）"]
        PG["Supabase pgvector\nOpenAI Embeddings\n已有基础设施·零新增"]
    end

    S1 -->|文档积累到 100+| S2
```

---

## 10. Skill 地图

| Skill | 状态 | 输入 | 输出 |
|-------|------|------|------|
| `/research` | ❌ 待建 | 问题域描述 | research-issues + 调研 comment |
| `/prd` | ❌ 待建 | research-issue(s) | master-issue body = PRD |
| `/batch-issue` | ❌ 待建 | master-issue #N | child-issues × M |
| `/issue` | 🔧 重构自 issule-goal | master-issue 上下文 | 单个 child-issue，AC+DoD |
| `/cursor` | ✅ 扩展 | `--label child` | 直接 commit + 报告写回 |
| `/issule-verify` | ✅ 扩展 | master AC + child 报告 | close all / re-queue |
| `/browser` | ✅ 复用 | URL | 页面内容（/research 内部） |
| `/gh` | ✅ 复用 | - | issue 全程读写 |

**待建优先级：** P1 `/batch-issue` → P2 `/prd` → P3 `/research` → P4 `/issue` 重构

---

## 架构决策记录

| # | 决策 | 结论 |
|---|------|------|
| D1 | 调研信息源 | /browser + /gh历史 + 用户输入 + 私有RAG |
| D2 | 调研到PRD过渡 | research-issue 独立类型，/prd 产出干净 master-issue |
| D3 | 单一契约 | master-issue = PRD，body+comments 贯穿始终 |
| D4 | 批量边界 | 只有 /batch-issue 派生 child-issues |
| D5 | child-issue 核心 | AC + DoD（替代 T1/T2/T3） |
| D6 | 验收机制 | /issule-verify skill，无 PR，直接 close |
| D7 | 知识库 | 阶段1 markdown，阶段2 Supabase pgvector |
| D8 | 人机确认点 | 4个强制 🔔，全记录在 master-issue comment |
| D9 | skill 命名 | /prd·/issue·/batch-issue，无系统冲突 |
