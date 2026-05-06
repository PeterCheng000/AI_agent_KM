# 06 — Multi-Agent：多個 Agent 怎麼協作？

> **核心問題**：單一 Agent 已經很強了，為什麼還需要多個 Agent？它們怎麼分工、怎麼溝通？

---

## 為什麼需要 Multi-Agent？

單一 Agent 有三道物理限制，不是靠「模型更聰明」就能解決的。

### 限制一：Context Window 有物理上限

```
任務：「分析 500 份客戶合約，找出所有有爭議的條款」

500 份合約 ≈ 需要 400k tokens
目前最大 context ≈ 128k~200k tokens

→ 直接塞不下
```

就算 context 夠大，費用隨長度**平方級成長**，而且模型對 context 中間部分的注意力會下降（Lost in the Middle 問題）。

### 限制二：一個 System Prompt 難以身兼多職

```
同一個任務需要：
  角色A：法律專家    → 理解條款的法律含義
  角色B：資料分析師  → 從 500 份找出規律
  角色C：報告撰寫者  → 寫出結構清晰的報告
  角色D：品質審查員  → 確認報告沒有錯誤

一個 Agent 的 System Prompt 只有一個
→ 很難同時具備所有專業，容易「角色混亂」
```

### 限制三：只能串行執行

```
單一 Agent：
合約1 → 合約2 → 合約3 → ... → 合約500（依序）

Multi-Agent：
Agent A → 合約 1~125   ┐
Agent B → 合約 126~250 ┤ → 同時跑 → 時間縮短 4 倍
Agent C → 合約 251~375 ┤
Agent D → 合約 376~500 ┘
```

---

## 用人類組織類比

```
Single Agent：一個很強的全端工程師，自己查資料、自己寫、自己測
→ 適合小任務，一個人能掌握全局

Multi-Agent：一間軟體公司
→ PM 規劃、工程師開發、QA 測試、法務審查
→ 每人專注自己領域，PM 協調
→ 能處理一個人根本完成不了的規模
```

**本質轉變：從「個人能力的極限」跨越到「組織協作的規模」。**

---

## Multi-Agent 的三種架構模式

### 模式一：Orchestrator-Worker（最常見）

```
使用者
  │
  ▼
Orchestrator Agent（PM）
  │ 負責：拆解任務、分派、收集結果、整合輸出
  │ 不做具體工作
  ├─────────────────┐
  │                 │
Worker A         Worker B         Worker C
（法律分析）      （資料提取）     （報告撰寫）
```

**特性**：
- 清楚的階層關係
- Orchestrator 是單點，容易追蹤任務狀態
- 適合任務能被清楚拆解的場景

**LangGraph 的實作方式**：定義 Orchestrator 為 Supervisor Node，Worker 為各個子 Graph。

---

### 模式二：Pipeline（流水線）

```
Raw Data
   ↓
Agent A（清洗資料）
   ↓
Agent B（分析資料）
   ↓
Agent C（生成報告）
   ↓
Agent D（品質審查）
   ↓
最終輸出
```

**特性**：
- 每個 Agent 的輸出是下一個的輸入
- 流程固定，可預測
- 適合有明確先後順序的任務
- 每個 Agent 只需要知道自己的輸入和輸出格式

---

### 模式三：Peer-to-Peer（對等協作）

```
Agent A ←──────→ Agent B
   ↑                 ↑
   │                 │
   └──── Agent C ────┘
```

**特性**：
- 沒有固定階層，Agent 之間平等溝通
- 適合需要辯論、互相審查的場景
- 例：讓兩個 Agent 分別提出方案，再讓第三個 Agent 評判

**代表性應用**：
- 一個 Agent 提出論點，另一個找漏洞
- 一個 Agent 寫程式碼，另一個審查和測試

---

## Agent 之間怎麼溝通？

### 方式一：共享狀態（Shared State）

```python
# 所有 Agent 都讀寫同一個 State 物件
state = {
    "task": "分析合約",
    "contracts": [...],
    "analysis_results": {},  # Agent A 寫入
    "report_draft": "",      # Agent B 寫入
    "review_comments": [],   # Agent C 寫入
    "final_report": ""       # 最終輸出
}
```

**優點**：簡單直觀，所有 Agent 都能看到全局
**缺點**：可能有並發寫入衝突

### 方式二：訊息傳遞（Message Passing）

```
Agent A 完成後，發送訊息給 Agent B：
{
  "from": "agent_a",
  "to": "agent_b",
  "type": "task_complete",
  "payload": { "result": "..." }
}
```

**優點**：清楚的責任邊界，適合分散式系統
**缺點**：需要設計訊息格式和路由

---

## 實際架構範例

### 任務：「研究一家公司，生成投資分析報告」

```
用戶：「幫我分析台積電，出一份投資報告」
   │
   ▼
Orchestrator：
  分析任務 → 拆成 4 個子任務

   ┌──────────────┬──────────────┬──────────────┐
   ▼              ▼              ▼              ▼
Research        Financial      News           Risk
Agent           Agent          Agent          Agent
────────        ─────────      ─────          ──────
搜尋公司基本    分析財務數字    搜尋近期新聞    評估風險因子
資料            EPS、本益比     競爭對手動態    產業趨勢

   └──────────────┴──────────────┴──────────────┘
                          │
                          ▼
                   Orchestrator 收集結果
                          │
                          ▼
                   Writer Agent
                   整合所有資訊，撰寫報告
                          │
                          ▼
                   Review Agent
                   審查報告完整性和準確性
                          │
                          ▼
                   最終報告輸出給用戶
```

---

## 代表性框架比較

| 框架 | 主要架構模式 | 特色 | 適合場景 |
|------|------------|------|---------|
| **LangGraph** | 狀態機（State Machine） | 精細控制流程、支援 Breakpoint | 需要可控、有條件分支的流程 |
| **AutoGen**（微軟） | Peer-to-Peer | 多 Agent 對話、辯論 | 需要多角度討論的任務 |
| **CrewAI** | Orchestrator-Worker | 角色定義清楚、易上手 | 快速建立有分工的 Agent 團隊 |
| **LangChain** | 各種都有 | 生態最大、模組最豐富 | 通用開發 |

---

## Multi-Agent 的挑戰

### 挑戰一：協調複雜度
Agent 越多，溝通路徑越多，越難追蹤問題出在哪。

```
解法：
- 清楚定義每個 Agent 的責任邊界
- 每個 Agent 的輸入輸出格式要嚴格規範
- 加入日誌追蹤每個步驟（LangSmith 等工具）
```

### 挑戰二：錯誤傳播
Agent A 的錯誤輸出可能讓 Agent B 整個跑偏。

```
解法：
- 每個 Agent 加入輸出驗證
- 關鍵節點加入 Verification Agent
- 設計 Retry 機制
```

### 挑戰三：成本爆炸
多個 Agent 同時跑 = 多個 LLM 呼叫 = 費用倍增。

```
解法：
- Orchestrator 用強模型，Worker 用便宜模型
- 能平行就平行，減少串行等待
- 快取重複查詢的結果
```

---

## 什麼時候用 Multi-Agent？

```
任務可以清楚拆解成獨立子任務？
  → Yes → 考慮 Multi-Agent

子任務可以平行執行？
  → Yes → 效益更大

任務需要不同專業角色？
  → Yes → 各角色設計獨立 Agent

─────────────────────────
單一任務、流程簡單？
  → 先用 Single Agent
  → 等真的碰到瓶頸再考慮 Multi-Agent
```

> **工程原則**：Multi-Agent 帶來複雜度，不要過早引入。先用最簡單的方式解決問題。

---

## 相關文件

- [01 — 演化脈絡](01_演化脈絡.md)：Multi-Agent 為什麼是演化的下一步
- [02 — Agent 解剖](02_Agent解剖.md)：單一 Agent 的架構
- [08 — 工程挑戰](08_工程挑戰.md)：Multi-Agent 的成本控制和可靠性
- [09 — 未解問題](09_未解問題.md)：Multi-Agent 協作模式目前還沒有定論
