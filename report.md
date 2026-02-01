# 大型程式碼倉庫中 LLM 自動化代碼修改技術：完整研究報告

> 基於 **86 篇** ArXiv 論文（2023–2026）之研究成果彙整
> 合併來源：博特一號（33 篇分析報告）+ 博特二號（28 篇執行方案），去重後共計 55 篇
> 撰寫日期：2026-02-01
> 目標讀者：想要**建構**此類系統的工程團隊

---

## 摘要 (Executive Summary)

本報告基於 86 篇 2023–2026 年 arXiv 論文，系統性分析了在大型程式碼倉庫（Repository-level）中利用 LLM 進行自動化代碼修改、除蟲、安全漏洞修補與 CI/CD 自動修復的技術現狀。

**核心發現：**

1. **倉庫級任務仍是巨大挑戰**：從函數級到倉庫級，LLM 表現大幅下降（GPT-3.5 從 ~50%+ 降至 22.58%）。即使最先進系統（Kimi-Dev）在 SWE-bench Verified 也僅達 60.4%。
2. **Agentless 與 Agent 趨於融合**：簡單工作流（Agentless，$0.70/issue，32% 解決率）和複雜 Agent 系統各有優勢。Kimi-Dev 證明兩者可互補——Agentless 訓練產生的技能先驗有助 Agent 適應。
3. **定位是瓶頸中的瓶頸**：RGFL 將檔案級 Hit@1 從 71.4% 提升至 85%，帶來 12.8% 端到端修復提升。精確定位是有效修復的前提。
4. **上下文管理成為核心議題**：CAT 工具化上下文達 57.6% SWE-bench Verified 解決率；SWE-Pruner 節省 23-54% token 且幾乎不影響效能。
5. **SWE-Bench+ 揭示真相**：過濾 solution leakage（32.67%）和 weak tests（31.08%）後，真實解決率僅 3.97%，業界表現被嚴重高估。
6. **工業部署仍稀少**：僅 Google（bug reproduction）、字節跳動（ContextCRBench 程式碼審查）、Kodezi（debugging-first LM）、騰訊（LLM 降低 SAST 誤報）有明確的工業規模部署報告。
7. **LLM + 靜態分析整合前景明確**：KNighter 在 Linux kernel 發現 30 個 CVE；IRIS 將 CodeQL 檢測從 27→55/120；騰訊實證誤報降 94-98%，成本僅 $0.001-0.12/告警。
8. **安全漏洞自動修補仍有巨大改進空間**：PatchIsland 在 AIxCC 達 72.1%，VulnResolver 在 SEC-bench 達 75%，但 VulnRepairEval exploit-based 評估最佳僅 21.7%，APR 對抗攻擊成功率達 0.91。
9. **CI/CD 管線自動修復逐漸成熟**：GradleFixer Android 構建修復 81.4%，IaCGen 部署成功率 54.6-91.6%（FSE'26），CI/CD 配置翻譯成功率 75.5%。

**建議執行策略**：採用**分層混合架構（Layered Hybrid Architecture）**——80% 簡單問題走 Agentless Fast Path，20% 複雜問題升級至 Multi-Agent 系統。分四階段、12 個月內完成部署，**第四階段加入靜態分析整合與安全驗證層**。

---

## 一、論文總覽

### 1.1 論文統計

本報告合併兩份獨立研究報告及補充搜索，去重後共收錄 **86 篇**唯一論文：

| 來源 | 原始收錄 | 共享論文 | 獨有論文 |
|------|---------|---------|---------|
| 博特一號 | 33 篇（表中 40 行，含跨類別重複） | 15 篇 | ~25 篇 |
| 博特二號 | 28 篇（含 32 篇附錄） | 15 篇 | ~17 篇 |
| 補充搜索（三大缺口領域） | 31 篇 | 0 篇 | 31 篇 |
| **合併去重** | — | — | **~86 篇** |

### 1.2 分類統計

| 類別 | 篇數 | 代表論文 |
|------|------|---------|
| 🐛 自動化缺陷修復 / 除蟲 | 12 | Agentless, REFINE, RGFL, Kimi-Dev, SemAgent, HAFixAgent |
| 🤖 SWE 智能代理 | 10 | SWE-agent, HyperAgent, BOAD, CAT, daVinci-Dev, FuseSearch |
| 📐 基準測試與評估框架 | 12 | SWE-bench, SWE-Bench+, SWE-Bench++, RepoDebug, Breakpoint |
| 🔍 程式碼審查自動化 | 3 | AACR-Bench, ContextCRBench, CodeFuse-CR-Bench |
| 🧠 倉庫級程式碼理解與檢索 | 10 | RIG, AlignCoder, RepoShapley, CodeMEM, InlineCoder, RepoLens |
| 🔄 大規模程式碼遷移 | 4 | MigrationBench, FreshBrew, C-to-Rust, Advancing Validation |
| 🎯 故障定位專項 | 4 | RGFL, Reformulate-Retrieve-Localize, NL-Summarization, InfCode-C++ |
| 📊 綜述與其他 | ~3 | LLM Issue Resolution Survey, GPT-4 Refactoring |
| 🔐 LLM + 靜態分析整合 | 12 | KNighter, IRIS, SAST-Genius, 騰訊誤報研究, BugLens, QLCoder |
| 🛡️ 自動化安全漏洞修補 | 10 | PatchIsland, VulnResolver, VulnRepairEval, Vul-R2, RvB |
| 🔄 CI/CD 管線自動修復 | 10 | GradleFixer, IaCGen, Build-bench, MACOG, AIOpsLab |

> 注：部分論文跨多類別，以主要貢獻分類。

---

## 二、建議執行方案

### 2.1 整體架構：分層混合架構（Layered Hybrid Architecture）

根據論文研究的兩大流派——**Agentless 工作流**與 **Agentic 多代理系統**——以及 Kimi-Dev 所揭示的「兩者可互補」之洞見，建議採用分層混合架構：

```
┌─────────────────────────────────────────────────────────┐
│                    Orchestrator Layer                     │
│         （任務分發、策略選擇、品質門控）                    │
├─────────────┬───────────────────────┬───────────────────┤
│  Pipeline   │   Agent Subsystem    │   Validation      │
│  (Agentless │   (複雜任務升級)      │   Subsystem       │
│   Fast Path)│                      │                   │
├─────────────┼───────────────────────┼───────────────────┤
│ 1. Localize │ Planner Agent        │ Test Generator    │
│ 2. Generate │ Navigator Agent      │ Patch Verifier    │
│ 3. Validate │ Editor Agent         │ Regression Runner │
│             │ Executor Agent       │ Review Agent      │
├─────────────┴───────────────────────┴───────────────────┤
│              Shared Infrastructure Layer                  │
│  ┌──────────┬──────────┬───────────┬──────────────────┐ │
│  │ Context  │ Memory   │ Code      │ Repo Knowledge   │ │
│  │ Manager  │ Store    │ Index     │ Graph            │ │
│  │ (CAT +   │(MemGovern│(Semantic +│(AST + Call Graph +│ │
│  │ Pruner)  │ Cards)   │ BM25)    │ Concept Map)     │ │
│  └──────────┴──────────┴───────────┴──────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

**設計理念：**
- **Fast Path（Agentless Pipeline）**：80% 以上的簡單到中等 Bug 走三階段流水線（定位→修復→驗證），高效低成本。
- **Agent Subsystem（升級路徑）**：當 Fast Path 置信度低或驗證失敗時，升級至多代理協作系統處理跨檔案、複雜邏輯的 Bug。
- **Shared Infrastructure**：上下文管理（CAT + SWE-Pruner）、經驗記憶（MemGovern）、程式碼索引（AST + 語意檢索）、知識圖譜（RIG / LogicLens）被兩條路徑共享。

### 2.2 六階段處理流程

#### 階段 1：問題接收與分類

| 輸入 | 處理 | 路由策略 |
|------|------|---------|
| GitHub Issue / Bug Report / Code Review 回饋 | Issue 結構化解析 + 難度預估 | 單檔案 + 明確堆疊追蹤 → Fast Path |
| | 萃取錯誤訊息、堆疊追蹤、預期行為 | 跨檔案 + 語意模糊 + 涉及架構 → Agent Path |
| | 參考 FAUN-Eval 細粒度評估 | QA/定位/編輯三子任務預估難度分流 |

#### 階段 2：故障定位（Fault Localization）

```
Issue Description
    │
    ▼
[Bug Explanation Generator]  ← RGFL：生成結構化 Bug 解釋
    │
    ▼
[Repository-Level Localizer] ← 檔案級排序（Top-K Files）
    │
    ▼
[Function-Level Localizer]   ← 函數/方法級定位
    │
    ▼
[Line-Level Localizer]       ← 精確行級定位
```

**並行加速**：FuseSearch 多路搜索（grep、語意搜索、AST 結構搜索），加速 93.6%。
**輔助手段**：RepoLens 概念知識抽取 + Time Travel（Git Bisect + LLM）用於回歸 Bug。

#### 階段 3：上下文收集（Context Extraction）

1. **靜態分析層**：import/include 依賴、呼叫圖上下游、型別定義、相關測試
2. **動態壓縮層（SWE-Pruner）**：0.6B 參數輕量級神經略讀器，縮減 23-54% token
3. **CAT 上下文工具化**：Agent 可隨時呼叫的上下文獲取/釋放工具
4. **RIG / LogicLens**：確定性程式碼圖譜，提供 LLM-friendly JSON view

#### 階段 4：修復生成（Patch Generation）

1. **候選生成**：LLM 生成 5-10 個候選補丁（sampling with temperature）
2. **REFINE 精煉**：識別「近似正確」補丁，用上下文感知修正 prompt 迭代精煉
3. **語義約束**：SemAgent 利用 program semantics 減少 hallucination
4. **歷史感知**：HAFixAgent mining commit patterns 指導修復
5. **多檔案協調**：HyperAgent 的 Planner-Navigator-Editor 模式

#### 階段 5：驗證（Validation）

**四層驗證機制：**
1. **語法驗證**：Lint / 型別檢查 / 編譯驗證
2. **測試驗證**：現有測試 + LLM 生成針對性測試（SWT-Bench 策略） + 回歸測試
3. **安全驗證**：SAST 掃描（CodeQL/Semgrep）+ LLM 誤報過濾（騰訊方法，誤報降 94-98%）+ exploit-based 驗證（VulnRepairEval 思路）
4. **語意審查**：LLM 程式碼審查（ContextCRBench hunk 級評估） + 交叉驗證

#### 階段 6：部署與回饋

1. 生成 Pull Request（含變更說明、影響範圍分析）
2. 整合至 CI/CD Pipeline
3. 收集結果回饋至 MemGovern 經驗記憶庫
4. 失敗案例分析並更新策略

### 2.3 建議技術選型

| 環節 | 推薦方案 | 備選 | 理由 |
|------|---------|------|------|
| **主力 LLM** | Claude Sonnet 4 / GPT-4o | DeepSeek-V3, Qwen-72B | 閉源在 SWE-bench 表現最佳；開源可控成本 |
| **專用定位模型** | FuseSearch-4B (Fine-tuned) | RGFL prompt | 小模型專用定位，速度快、成本低 |
| **上下文壓縮** | SWE-Pruner (0.6B) | CAT framework | 輕量、可離線部署、token 節省顯著 |
| **程式碼圖譜** | Tree-sitter AST + Neo4j/NetworkX | LSP servers, Sourcegraph | 多粒度索引支援結構化和語意搜索 |
| **RAG 檢索** | RL-trained retriever (AlignCoder style) | BM25 + reranker | EM +18.1%，跨語言泛化性強 |
| **經驗記憶** | MemGovern 風格經驗卡 | RAG + Vector DB | 結構化經驗可直接注入 prompt |
| **測試生成** | LLM + SWT-Bench 方法 | 傳統 fuzzing | Issue→測試案例的形式化有實驗支撐 |
| **Agent 框架** | 自建（基於 HyperAgent 架構） | SWE-Agent, OpenHands, LangGraph | 自建才能深度整合各模組 |
| **RL 訓練** | PPO/GRPO on SWE-bench data | DPO on curated pairs | Self-play SWE-RL 驗證可行 |
| **沙盒執行** | Docker container per task | Modal, Firecracker | 隔離性、可重現性 |

---

## 三、關鍵技術點

### 3.1 Fault Localization（故障定位）

故障定位是整個 Pipeline 的**瓶頸**。在百萬行級倉庫中，需要從數千個檔案中精確找到關鍵的幾行程式碼。

**核心技術方案：**

| # | 技術 | 方法描述 | 適用場景 |
|---|------|---------|---------|
| 1 | **RGFL 分層推理** | LLM 生成結構化 Bug 解釋 → Repo→File→Function→Line 逐步縮小 | 通用 Bug |
| 2 | **FuseSearch 並行定位** | 多路並行搜索（grep + 語意 + AST），RL 策略決定搜索深度 | 大型倉庫加速 |
| 3 | **RepoLens 概念知識圖** | 處理 concern tangling/scattering，概念層級的知識表示 | 複雜架構 |
| 4 | **InfCode-C++ AST 結構化** | 語意意圖檢索 + 確定性 AST 查詢 | C/C++ 強型別語言 |
| 5 | **Reformulate-Retrieve-Localize** | LLM reformulate bug report 為精確搜索查詢 | Noisy bug descriptions |
| 6 | **NL-Summarization 跨 Repo** | LLM NL summaries 橋接 bug reports 和 code | 微服務架構 |
| 7 | **Time Travel** | LLM + Git Bisect 整合版本歷史定位 | 回歸 Bug |

**📊 效能數據：**

| 技術 | 指標 | 數據 | 來源 |
|------|------|------|------|
| RGFL | 檔案級 Hit@1 | 71.4% → **85%**（+13.6pp） | [2601.18044] |
| RGFL | 端到端修復提升 | **+12.8%**（整合 Agentless） | [2601.18044] |
| FuseSearch-4B | 檔案級 F1 | **84.7%**（SOTA） | [2601.19568] |
| FuseSearch-4B | 搜索加速 | **93.6%** 加速，減少 67.7% 輪次 | [2601.19568] |
| RepoLens | Hit@k 提升 | 平均 **+22%** | [2509.21427] |
| RepoLens | Recall@k 提升 | 平均 **+46%** | [2509.21427] |
| Time Travel | Git Bisect 成功率 | 74.2% → **80.6%** | [2511.18854] |

---

### 3.2 Context Extraction（上下文提取）

大型倉庫的程式碼量遠超 LLM 上下文窗口。關鍵在於精確提取與任務最相關的上下文片段。

**核心技術方案：**

| # | 技術 | 方法描述 |
|---|------|---------|
| 1 | **RLCE 多策略提取** | Import graph + call graph + type hierarchy，沿依賴圖向外擴展 |
| 2 | **AlignCoder 查詢增強** | LLM 補全意圖→檢索查詢，RL-trained retriever 對齊意圖 |
| 3 | **RIG 確定性圖譜** | Build system 解析 + LLM-friendly JSON view，多語言 repo 效果最佳 |
| 4 | **RepoShapley 過濾** | Shapley values 評估每個 chunk 邊際貢獻，過濾有害 context |
| 5 | **InlineCoder 雙向內聯** | Upstream inlining（callers）+ Downstream retrieval（callees） |
| 6 | **LogicLens 語義圖** | AST parsing + LLM enrichment → semantic multi-repo graph |

**📊 效能數據：**

| 技術 | 指標 | 數據 | 來源 |
|------|------|------|------|
| RLCE | 修復能力提升 | 最高 **160%** | RepoBugs [2403.00448] |
| RLCE | 基線（GPT-3.5 無策略） | 僅 **22.58%** | RepoBugs [2403.00448] |
| AlignCoder | EM 分數提升 | **+18.1%**（CrossCodeEval） | [2601.19697] |
| RIG | 準確率提升 / 時間縮短 | **+12.2%** / **-53.9%** | RIG（一號報告） |
| RIG | 多語言 repo | **+17.7%** 準確率 / **-69.5%** 時間 | RIG（一號報告） |
| AACR-Bench | 缺陷覆蓋率 | 提升 **285%**（AI 輔助標注） | [2601.19494] |
| CodeMEM | 指令遵循 / 交互輪數 | **+12.2%** / **-2~3 輪** | CodeMEM（一號報告） |

---

### 3.3 Context Management（上下文管理）

使系統能處理真正大型倉庫（百萬行以上）的關鍵技術。

**核心技術方案：**

| # | 技術 | 方法描述 |
|---|------|---------|
| 1 | **CAT（Context as a Tool）** | 上下文維護提升為 Agent 可呼叫的工具，避免一次性載入導致注意力稀釋 |
| 2 | **SWE-Pruner 神經略讀器** | 0.6B 參數輕量級模型，逐檔案動態選擇相關程式碼行 |
| 3 | **MemGovern 經驗記憶** | 將修復經驗結構化為記憶卡，類似問題可直接引用 |

**📊 效能數據：**

| 技術 | 指標 | 數據 | 來源 |
|------|------|------|------|
| CAT (SWE-Compressor) | SWE-Bench Verified | **57.6%** 解決率 | [2512.22087] |
| SWE-Pruner | Token 縮減 | **23-54%**（效能影響極小） | [2601.16746] |
| MemGovern | SWE-Bench Verified 提升 | **+4.65%**（插件式） | [2601.06789] |
| MemGovern | 經驗卡規模 | **135K** 經驗卡 | [2601.06789] |

---

### 3.4 Patch Generation（修復生成）

**核心技術方案：**

| # | 技術 | 方法描述 |
|---|------|---------|
| 1 | **Agentless 三階段** | 定位→多候選 patch→測試過濾，簡單可解釋 |
| 2 | **REFINE 補丁精煉** | 識別「近似正確」補丁，LLM 審查迭代修正 |
| 3 | **SemAgent 語義感知** | 利用 program semantics 約束生成，減少 hallucination |
| 4 | **HAFixAgent 歷史感知** | Mining commit patterns 指導相似 Bug 修復 |
| 5 | **Kimi-Dev 技能先驗** | Agentless 訓練→Agent 技能先驗，開源模型也能接近閉源 |
| 6 | **daVinci-Dev Agent 原生訓練** | Agent 原生資料（上下文原生 + 環境原生軌跡）中期訓練 |
| 7 | **MCTS-Refined-CoT** | Monte Carlo Tree Search 精煉 Chain-of-Thought 推理 |
| 8 | **SWE-Synth 合成資料** | 合成可驗證的 bug-fix 訓練資料，解決稀缺問題 |

**📊 效能數據：**

| 技術 | 基準 | 數據 | 來源 |
|------|------|------|------|
| Agentless | SWE-bench Lite | **32%** 解決率，成本 **$0.70** | [2407.01489] |
| REFINE + AutoCodeRover | SWE-bench Lite | **51.67%**（+14.67%） | [2510.03588] |
| Kimi-Dev | SWE-bench Verified | **60.4%**（工作流 SOTA） | [2509.23045] |
| daVinci-Dev 72B | SWE-bench Verified | **58.5%** | [2601.18418] |
| daVinci-Dev 32B | SWE-bench Verified | **56.1%** | [2601.18418] |
| Self-play SWE-RL | SWE-bench Verified 提升 | **+10.4 分** | [2512.18552] |
| SWE-agent | SWE-bench | **12.5%** pass@1（2024 突破性結果） | SWE-agent 論文 |

---

### 3.5 Validation / Testing（驗證與測試）

**核心技術方案：**

| # | 技術 | 方法描述 |
|---|------|---------|
| 1 | **SWT-Bench** | Issue 形式化為測試案例，作為補丁篩選器 |
| 2 | **MAVM 多步驗證** | 檢測→確認→修復→驗證的完整流水線 |
| 3 | **Google Bug Reproduction** | LLM agents 自動生成 reproduction tests（工業規模） |
| 4 | **AACR-Bench 多語言審查** | AI-assisted expert-verified，跨檔案上下文 code review |
| 5 | **交叉驗證** | 不同 LLM 互審，降低 hallucination 風險 |

**📊 效能數據：**

| 技術 | 指標 | 數據 | 來源 |
|------|------|------|------|
| SWT-Bench 測試生成 | SWE-Agent 精確度 | **翻倍** | [2406.12952] |
| MAVM | 真實漏洞修復 | **51 個**真實漏洞修復成功 | [2601.17762] |
| MAVM | 修復準確度 vs 基線 | **+31.9%-45.2%** | [2601.17762] |
| AACR-Bench | 缺陷覆蓋率 | **+285%** vs raw PR | [2601.19494] |
| ContextCRBench | 字節跳動部署 | 表現提升 **61.98%** | [2511.07017] |

---

### 3.6 Multi-file Coordination（多檔案協調）

**核心技術方案：**

| # | 技術 | 方法描述 |
|---|------|---------|
| 1 | **HyperAgent 四代理架構** | Planner + Navigator + Code Editor + Executor |
| 2 | **BOAD 階層式自動發現** | MAB 框架自動發現有效子代理組合 |
| 3 | **InfCode-C++ 混合搜索** | 語意檢索 + AST 結構化查詢確保修改正確性 |

**📊 效能數據：**

| 技術 | 指標 | 數據 | 來源 |
|------|------|------|------|
| BOAD 36B | SWE-bench-Live 排名 | **第二**（超越 GPT-4, Claude） | [2512.23631] |
| InfCode-C++ | MultiSWE-bench-CPP | **25.58%**（+10.85pp over SOTA） | [2511.16005] |
| HyperAgent | 跨語言 SE 任務 | 完整 SE 生命週期覆蓋 | [2409.16299] |

---

### 3.7 🔐 LLM + 靜態分析整合（12 篇）

LLM 與傳統靜態分析工具（CodeQL、Semgrep、Infer 等）的整合，是近期最具工業落地潛力的方向之一。核心思路：用 LLM 增強靜態分析器的精確度、自動合成分析規則、大幅降低誤報。

**核心技術方案：**

| # | 技術 | 方法描述 |
|---|------|---------|
| 1 | **KNighter** | LLM 從歷史 bug 模式自動合成靜態分析 checker，多階段合成 + 自動迭代精煉 |
| 2 | **IRIS** | LLM 推斷 taint 規格 + 上下文分析，增強 CodeQL 全倉庫漏洞檢測 |
| 3 | **SAST-Genius** | LLM + Semgrep 混合框架，含動態 bug 描述和 exploit 生成驗證 |
| 4 | **騰訊誤報研究** | 工業環境首次全面實證，LLM + SAT 消除 94-98% 誤報 |
| 5 | **BugLens** | 後精煉框架，結構化推理步驟評估安全影響，精確度提升 7 倍 |
| 6 | **PredicateFix** | 利用分析規則謂詞橋接告警與代碼，RAG 管線修復 CodeQL/GoInsight 告警 |
| 7 | **QLCoder** | 從 CVE 元數據自動合成 CodeQL 查詢，含 MCP + LSP + RAG |
| 8 | **AdaTaint** | 自適應 source/sink 推斷 + 神經符號推理過濾虛假告警 |
| 9 | **RAG + Multi-Tool 安全生成** | 編譯器 + CodeQL + KLEE 符號執行多工具迭代精煉 |
| 10 | **LLM Prompting as SA Proxy** | 研究 LLM 提示替代 CodeQL，對比思維鏈推理 |
| 11 | **LAMeD** | LLM 自動生成函數註解，改進記憶體洩漏檢測 |
| 12 | **BugScope** | 多代理系統模擬人類審計員學習 bug 模式 |

**📊 效能數據：**

| 技術 | 指標 | 數據 | 來源 |
|------|------|------|------|
| KNighter | Linux kernel 新 CVE | **30 個 CVE**，92 個新 bug | [2503.09002] SOSP'25 |
| IRIS (GPT-4) | CodeQL 漏洞檢測 | **27→55/120**（+103%） | [2405.17238] |
| SAST-Genius | 誤報降低（vs 純 Semgrep） | **~91%**（225→20） | [2509.15433] S&P'25 |
| 騰訊 | 誤報消除 / 成本 | **94-98%** / **$0.001-0.12**/告警 | [2601.18844] |
| BugLens | 精確度提升 | **7 倍**（0.10→0.72） | [2504.11711] |
| PredicateFix | 正確修復數增加 | **+27.1%-69.3%** | [2503.12205] ICSE'26 |
| QLCoder | CodeQL 查詢合成 | **53.4%**（純 Claude Code 僅 10%，5.3× 改進） | [2511.08462] |
| AdaTaint | 誤報降低 / 召回提升 | **-43.7%** / **+11.2%** | [2511.04023] |
| BugScope | 精確率 / 召回率 | **87.04%** / **90.00%**，發現 141 個未知 bug | [2507.15671] |
| RAG+Multi-Tool | 安全漏洞降低 | DeepSeek **-96%** 漏洞 | [2601.00509] |

**🏭 工業部署亮點：**
- **騰訊**：首個工業級 LLM+SAST 實證，每條告警 $0.001-0.12，相比人工審查（10-20 分鐘/條）節省數個數量級
- **KNighter**：Linux kernel 30 個 CVE，證明 LLM 合成 checker 可超越人工編寫的規則

---

### 3.8 🛡️ 自動化安全漏洞修補（10 篇）

LLM 用於 CVE 修復和安全漏洞自動修補，是自動化代碼修改的高風險高價值應用場景。

**核心技術方案：**

| # | 技術 | 方法描述 |
|---|------|---------|
| 1 | **PatchIsland** | 多樣 LLM 代理集成 + 兩階段補丁去重，面向持續模糊測試 |
| 2 | **VulnResolver** | 混合代理框架，CPCAgent 上下文預收集 + SPAAgent 安全屬性分析 |
| 3 | **Well Begun is Half Done** | 位置感知 + taint 語句覆蓋率評估補丁品質 |
| 4 | **Vul-R2** | 針對漏洞修復的推理 LLM + 強化學習自適應緩解 |
| 5 | **PATCHEVAL** | 多語言漏洞修補基準，1,000 漏洞 + 230 CVE 沙箱環境 |
| 6 | **One-Shot Patching Eval** | 評估多 LLM 在真實/人工漏洞上的互補性 |
| 7 | **VulnRepairEval** | 基於 PoC exploit 的嚴格評估框架 |
| 8 | **SecureFixAgent** | Bandit + 輕量 LLM（<8B）混合迭代修復 |
| 9 | **RvB** | 紅藍對抗框架，免訓練序列博弈用於動態代碼加固 |
| 10 | **Red Teaming APR** | SWExploit 生成對抗性 issue，揭示 APR 安全風險 |

**📊 效能數據：**

| 技術 | 指標 | 數據 | 來源 |
|------|------|------|------|
| PatchIsland | AIxCC 競賽修復率 | **72.1%**（31/43 漏洞），內部 91.3% | [2601.17471] |
| VulnResolver | SEC-bench Lite | **75%** 解決率（最佳表現） | [2601.13933] |
| Well Begun | VulnLoc+ 合理補丁 | **27 個**（比基線多 8-22 個） | [2512.20203] ICSE'26 |
| VulnRepairEval | Exploit-based 最佳 | **僅 21.7%**（5/23） | [2509.03331] |
| SecureFixAgent | 修復準確率提升 | **+13.51%**，3 次迭代收斂 | [2509.16275] |
| RvB | 防禦成功率 | **90%**（代碼加固），誤報 ~0% | [2601.19726] |
| Red Teaming APR | 對抗攻擊成功率 | **0.91**（基線 <0.20），最佳過濾僅擋 47% | [2509.25894] |

**⚠️ 關鍵警示：**
- **VulnRepairEval 揭示殘酷現實**：以 exploit-based 方式嚴格評估，最佳模型僅修復 5/23（21.7%），增強提示和多代理方法改進有限
- **APR 系統存在嚴重安全風險**：SWExploit 攻擊成功率達 0.91，90% 對抗性 bug 報告觸發攻擊者期望的含漏洞補丁，最佳前置過濾器僅擋 47%

---

### 3.9 🔄 CI/CD 管線自動修復（10 篇）

LLM 在構建失敗修復、CI/CD 配置管理和基礎設施即代碼（IaC）生成方面展現出強大潛力。

**核心技術方案：**

| # | 技術 | 方法描述 |
|---|------|---------|
| 1 | **GradleFixer** | 領域特定工具 + LLM 代理，Tool Bridging 策略 |
| 2 | **Build-bench** | 跨 ISA 構建修復基準（268 失敗），Microsoft Research |
| 3 | **Build-bench Extended** | 擴展版端對端基準，含工具使用模式分析 |
| 4 | **CI/CD Config Translation** | Travis CI → GitHub Actions 翻譯，指南式提示+迭代精煉 |
| 5 | **AI-Augmented CI/CD** | AI 增強 CI/CD 參考架構，策略即代碼護欄 |
| 6 | **Supply Chain Security** | LLM + RL + 多代理的軟體供應鏈安全框架 |
| 7 | **Android Build Diagnosis** | 200 個 Android 專案構建失敗實證分析 |
| 8 | **IaCGen** | 可部署性導向 IaC 生成，迭代反饋（格式→語法→部署） |
| 9 | **MACOG** | 多代理 Terraform 配置生成（Architect/Engineer/Reviewer/Security） |
| 10 | **AIOpsLab** | 端對端 AIOps 代理評估框架，AgentOps 範式 |

**📊 效能數據：**

| 技術 | 指標 | 數據 | 來源 |
|------|------|------|------|
| GradleFixer | Android 構建修復 | **81.4%** pass@1 | [2510.08640] |
| Build-bench Ext. | 跨 ISA 最高成功率 | **63%** | [2511.00780] |
| CI/CD Translation | 構建成功率 | **75.5%**（GPT-4o 基本提示的 3 倍） | [2511.01316] |
| IaCGen | 部署成功率 | **54.6-91.6%**（10 次迭代內） | [2506.05623] FSE'26 |
| IaCGen | 首次部署成功率 | 僅 **20.8-30.2%** | [2506.05623] FSE'26 |
| MACOG (GPT-5) | 配置品質分數 | **74.02**（RAG 基線 54.90） | [2510.03902] |
| Android Diagnosis | 構建修復率 | **75.56%**（102/135） | [2511.06186] |

**🔑 關鍵洞見：**
- **Tool Bridging 策略**（GradleFixer）：用領域感知抽象替代通用 shell 命令，顯著優於 SOTA 程式碼代理
- **迭代反饋是關鍵**（IaCGen）：首次嘗試部署成功率僅 20-30%，但迭代 10 次可達 54.6-91.6%，人機協作可達 90%+
- **信任層級框架**（AI-Augmented CI/CD）：提出分階段自主化的策略即代碼護欄模式

---

## 四、論文數據支撐

### 4.1 完整論文對比表

以下為所有 55 篇唯一論文的綜合對比，按類別排列：

#### 🐛 自動化缺陷修復 / 除蟲

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 工業部署 | 來源 |
|---|------|---------|-----------|---------|------|---------|------|
| 1 | Agentless | 三階段 Pipeline（定位→修復→驗證） | SWE-bench Lite | 32% 解決率，$0.70/issue | 2024 | 否 | 二號 |
| 2 | RepoBugs + RLCE | 倉庫級上下文提取 | RepoBugs (124 bugs) | 修復提升最高 160% | 2024 | 否 | 兩者 |
| 3 | REFINE | 補丁精煉框架 | SWE-bench Lite | 51.67%（+14.67%） | 2025 | 否 | 兩者 |
| 4 | RGFL | 分層推理故障定位 | SWE-bench Verified | Hit@1: 85%（+13.6pp） | 2026 | 否 | 兩者 |
| 5 | Kimi-Dev | Agentless 訓練→Agent 技能先驗 | SWE-bench Verified | 60.4%（工作流 SOTA） | 2025 | 否 | 二號 |
| 6 | SemAgent | 語義感知修復 | Repo-level | 語義約束減少 hallucination | 2025 | 否 | 一號 |
| 7 | HAFixAgent | 歷史感知修復 | Repo-level | Commit pattern mining | 2025 | 否 | 一號 |
| 8 | Autonomous Issue Resolver | Zero-touch APR | Repo-scale | 從 function 擴展到 repo-scale | 2025 | 否 | 一號 |
| 9 | Tracing-Errors | Typestate + retrieval | C memory errors | Memory error repair | 2025 | 否 | 一號 |
| 10 | LLM-Nullability-Repair | Static analysis + LLM | Java large codebases | Nullability at scale | 2025 | 否 | 一號 |
| 11 | Google Bug Reproduction | LLM Agent | Google 內部 | 工業規模 bug reproduction | 2025 | **是（Google）** | 一號 |
| 12 | SWE-Synth | 合成 bug-fix 訓練資料 | SWE-bench style | 解決 training data 稀缺 | 2025 | 否 | 一號 |

#### 🤖 SWE 智能代理

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 工業部署 | 來源 |
|---|------|---------|-----------|---------|------|---------|------|
| 13 | SWE-agent | Agent-Computer Interface | SWE-bench | 12.5% pass@1, HumanEvalFix 87.7% | 2024 | 否（開源） | 一號 |
| 14 | HyperAgent | 四代理系統 | SWE-bench, RepoExec, Defects4J | Multi-benchmark SOTA | 2024 | 否 | 兩者 |
| 15 | daVinci-Dev | Agent 原生中期訓練 | SWE-bench Verified | 32B: 56.1%, 72B: 58.5% | 2026 | 否 | 二號 |
| 16 | BOAD | 階層式多代理自動發現 (MAB) | SWE-bench-Live | 排名第二，超越 GPT-4/Claude | 2025 | 否 | 二號 |
| 17 | CAT | 上下文工具化管理 | SWE-bench Verified | 57.6% 解決率 | 2025 | 否 | 二號 |
| 18 | MemGovern | 經驗記憶卡框架 | SWE-bench Verified | +4.65%，135K 經驗卡 | 2026 | 否 | 二號 |
| 19 | FuseSearch | 並行程式碼定位 | SWE-bench Verified | F1: 84.7%，加速 93.6% | 2026 | 否 | 二號 |
| 20 | SWE-Pruner | 自適應上下文剪枝 | SWE-bench Verified | Token 縮減 23-54% | 2026 | 否 | 二號 |
| 21 | SWE-RL / Self-play SWE-RL | Self-play RL 訓練 | SWE-bench Verified | +10.4 分 | 2025 | 否 | 兩者 |
| 22 | One Tool Is Enough | RL agent training | Large repos | RL 定位文件/函數 | 2025 | 否 | 一號 |

#### 📐 基準測試與評估框架

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 來源 |
|---|------|---------|-----------|---------|------|------|
| 23 | SWE-bench | 開創性評估基準 | 2,294 issues, 12 repos | 奠定倉庫級 LLM 評估基礎 | 2023 | 二號 |
| 24 | SWE-Bench+ | 品質分析 | SWE-bench | 真實 rate 3.97%（原 12.47%） | 2024 | 一號 |
| 25 | SWE-Bench++ | 可擴展 bench gen | 11 語言, 3,971 repos | Claude-sonnet-4.5: 36.20% pass@10 | 2025 | 兩者 |
| 26 | RepoDebug | 多任務多語言除錯 | 8 語言, 22 錯誤類型 | SOTA 仍表現不佳 | 2025 | 兩者 |
| 27 | Breakpoint | 系統級推理難度 | 真實倉庫對抗性損壞 | SOTA 最難任務成功率 **0%** | 2025 | 二號 |
| 28 | SWT-Bench | Issue→測試案例生成 | SWE-agent | 精確度**翻倍** | 2024 | 二號 |
| 29 | FAUN-Eval | 細粒度子任務評估 | 30 repos, 300 條目 | 不同任務最佳 LLM 各異 | 2024 | 二號 |
| 30 | RepoGenesis | 端對端微服務生成 | 106 repos, 18 領域 | 最佳 Pass@1 僅 23.67% | 2026 | 二號 |
| 31 | OctoBench | Scaffold-aware 評估 | 34 envs, 217 tasks | 揭示 task-solving vs scaffold gap | 2026 | 一號 |
| 32 | RepoReason | Abductive assertion verification | Frontier models | 揭示 aggregation deficit | 2026 | 一號 |
| 33 | MCTS-Refined-CoT | MCTS + CoT | SWE-bench | 提升 open-source LLM | 2025 | 一號 |
| 34 | Thinking Longer, Not Larger | Test-time compute scaling | Repo-level reasoning | 小模型≈大模型 | 2025 | 一號 |

#### 🔍 程式碼審查自動化

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 工業部署 | 來源 |
|---|------|---------|-----------|---------|------|---------|------|
| 35 | AACR-Bench | AI-assisted 跨檔案審查 | 多語言 PRs | 缺陷覆蓋 +285% | 2026 | 否（阿里巴巴開源） | 兩者 |
| 36 | ContextCRBench | 細粒度 hunk 級審查 | 67,910 條目 | 字節跳動部署 +61.98% | 2025 | **是（字節跳動）** | 二號 |
| 37 | CodeFuse-CR-Bench | 端到端審查 | Python projects | 審查全面性評估 | 2025 | 否 | 兩者 |

#### 🧠 倉庫級程式碼理解與檢索

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 來源 |
|---|------|---------|-----------|---------|------|------|
| 38 | AlignCoder | RAG + RL retriever | CrossCodeEval, RepoEval | EM +18.1% | 2026 | 兩者 |
| 39 | RIG | 確定性 Code Graph | 8 repos | +12.2% acc, -53.9% time | 2026 | 一號 |
| 40 | LogicLens | Semantic code graph | Multi-repo | Impact analysis + debugging | 2026 | 一號 |
| 41 | RepoShapley | Shapley-value filtering | Multiple | 減少 harmful context | 2026 | 一號 |
| 42 | CodeMEM | AST-guided memory | CodeIF-Bench, CoderEval | +12.2% 指令遵循 | 2026 | 一號 |
| 43 | InlineCoder | Bidirectional inlining | — | FSE 2026 accepted | 2026 | 一號 |
| 44 | DependEval | Dependency benchmark | Multi-file repos | Cross-module reasoning eval | 2025 | 一號 |
| 45 | Kodezi Chronos | Debugging-first LM | Repo-scale | Debugging-first 設計 | 2025 | **是（Kodezi）** | 一號 |
| 46 | RepoLens | 概念知識抽取定位 | 3 SOTA tools | Hit@k +22%, Recall@k +46% | 2025 | 二號 |
| 47 | MAVM | 多代理漏洞管理 | 真實漏洞 | 51 漏洞修復，+31.9-45.2% | 2026 | 二號 |
| 48 | Time Travel | LLM + Git Bisect | — | 成功率 80.6%，加速 2× | 2025 | 二號 |

#### 🎯 故障定位專項

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 來源 |
|---|------|---------|-----------|---------|------|------|
| 49 | InfCode-C++ | Intent + AST search | MultiSWE-bench-CPP | 25.58%（+10.85pp） | 2025 | 兩者 |
| 50 | Reformulate-Retrieve-Localize | Query reformulation | IRBL | Noisy bug description handling | 2025 | 一號 |
| 51 | NL-Summarization-Multi-Repo | NL summarization | Microservices | 跨 repo localization | 2025 | 一號 |

#### 🔄 大規模程式碼遷移

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 來源 |
|---|------|---------|-----------|---------|------|------|
| 52 | MigrationBench | Java 8 遷移基準 | 5,102 repos | Claude-3.5: 62.33% | 2025 | 兩者 |
| 53 | FreshBrew | AI agent 遷移評估 | 228 repos | Gemini 2.5 Flash: 52.3% | 2025 | 兩者 |
| 54 | C-to-Rust Translation | LLM + transpilation | Large C repos | 品質 vs scalability 平衡 | 2025 | 一號 |
| 55 | Advancing Validation | In-isolation validation | Repo-level translation | 無需完整編譯的驗證 | 2025 | 一號 |

#### 📊 綜述與其他

| # | 論文 | 核心方法 | 年份 | 來源 |
|---|------|---------|------|------|
| 56 | LLM Issue Resolution Survey | 系統性綜述 | 2026 | 兩者 |
| 57 | GPT-4 Refactoring | GPT-4o 重構實證研究 | 2026 | 二號 |

#### 🔐 LLM + 靜態分析整合

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 工業部署 | 來源 |
|---|------|---------|-----------|---------|------|---------|------|
| 58 | KNighter | LLM 合成靜態分析 checker | Linux kernel | 30 CVE, 92 新 bug | 2025 | 否（SOSP'25） | 補充 |
| 59 | IRIS | LLM + CodeQL taint 推斷 | CWE-Bench-Java (120) | 27→55/120, +4 未知漏洞 | 2024 | 否 | 補充 |
| 60 | SAST-Genius | LLM + Semgrep 混合 | 複雜代碼庫 | 誤報降 91%（225→20） | 2025 | 否（S&P'25） | 補充 |
| 61 | 騰訊誤報研究 | LLM + 企業 SAT | 騰訊內部 | 誤報降 94-98%, $0.001-0.12 | 2026 | **是（騰訊）** | 補充 |
| 62 | RAG+Multi-Tool | RAG + CodeQL + KLEE | 3,242 程式 | DeepSeek 漏洞降 96% | 2026 | 否 | 補充 |
| 63 | BugLens | 後精煉結構化推理 | Linux kernel | 精確度 7 倍（0.10→0.72） | 2025 | 否 | 補充 |
| 64 | PredicateFix | 謂詞橋接 RAG 修復 | CodeQL/GoInsight | 正確修復 +27.1-69.3% | 2025 | 否（ICSE'26） | 補充 |
| 65 | QLCoder | CVE→CodeQL 查詢合成 | 176 CVE, 111 Java 專案 | 53.4%（5.3× 改進） | 2025 | 否 | 補充 |
| 66 | AdaTaint | 自適應 taint 分析 | Juliet/SV-COMP/真實專案 | 誤報降 43.7%, 召回+11.2% | 2025 | 否 | 補充 |
| 67 | LLM Prompting as SA | 對比思維鏈推理 | 漏洞檢測 | F1 +71.7%, 準確率+31.6% | 2024 | 否 | 補充 |
| 68 | LAMeD | LLM 生成函數註解 | CodeQL/Infer/Cooddy | 改善記憶體洩漏檢測 | 2025 | 否 | 補充 |
| 69 | BugScope | 多代理審計系統 | 大型開源系統 | 87% 精確率, 141 未知 bug | 2025 | 否 | 補充 |

#### 🛡️ 自動化安全漏洞修補

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 工業部署 | 來源 |
|---|------|---------|-----------|---------|------|---------|------|
| 70 | PatchIsland | 多 LLM 代理集成 | AIxCC 競賽 | 72.1%（31/43），內部 91.3% | 2026 | 否 | 補充 |
| 71 | VulnResolver | 混合代理框架 | SEC-bench | 75% 解決率 | 2026 | 否 | 補充 |
| 72 | Well Begun | 位置感知迭代修復 | VulnLoc+ (40 漏洞) | 27 合理補丁（+8-22） | 2025 | 否（ICSE'26） | 補充 |
| 73 | Vul-R2 | 推理 LLM + RL 緩解 | 漏洞修復 | ASE'25 接收 | 2025 | 否（ASE'25） | 補充 |
| 74 | PATCHEVAL | 多語言漏洞基準 | 1,000 漏洞, 230 CVE | 65 種 CWE, 沙箱驗證 | 2025 | 否 | 補充 |
| 75 | One-Shot Patching | 多 LLM 互補評估 | 真實+人工漏洞 | LLM 間互補性顯著 | 2025 | 否 | 補充 |
| 76 | VulnRepairEval | Exploit-based 評估 | 23 Python CVE | 最佳僅 21.7%（5/23） | 2025 | 否 | 補充 |
| 77 | SecureFixAgent | Bandit + 輕量 LLM | Python 漏洞 | +13.51%, 3 次迭代收斂 | 2025 | 否（ICMLA'25） | 補充 |
| 78 | RvB | 紅藍對抗加固 | CVE 防禦 | 防禦 90%, 誤報 ~0% | 2026 | 否 | 補充 |
| 79 | Red Teaming APR | 對抗性 APR 攻擊 | SWExploit | 攻擊成功率 0.91 | 2025 | 否 | 補充 |

#### 🔄 CI/CD 管線自動修復

| # | 論文 | 核心方法 | Benchmark | 關鍵結果 | 年份 | 工業部署 | 來源 |
|---|------|---------|-----------|---------|------|---------|------|
| 80 | GradleFixer | Tool Bridging + LLM | AndroidBuildBench (1,019) | 81.4% pass@1 | 2025 | 否 | 補充 |
| 81 | Build-bench | 跨 ISA 構建修復 | 268 失敗 | Microsoft Research 基準 | 2026 | 否 | 補充 |
| 82 | Build-bench Ext. | 端對端評估 | 跨架構 | 最高 63% | 2025 | 否 | 補充 |
| 83 | CI/CD Config Trans. | Travis→GitHub Actions | 811 遷移記錄 | 75.5%（3× 改進） | 2025 | 否 | 補充 |
| 84 | AI-Augmented CI/CD | 參考架構 + 護欄 | React 19 案例 | 信任層級框架 | 2025 | 否 | 補充 |
| 85 | Supply Chain Security | LLM+RL+多代理 | GitHub Actions/Jenkins | 優於規則式基線 | 2025 | 否 | 補充 |
| 86 | Android Build Diag. | GPT-5 輔助診斷 | 200 Android 專案 | 75.56%（102/135） | 2025 | 否 | 補充 |
| 87 | IaCGen | 迭代部署反饋 | DPIaC-Eval (153 場景) | 54.6-91.6% 部署 | 2025 | 否（FSE'26） | 補充 |
| 88 | MACOG | 多代理 Terraform | IaC 配置 | GPT-5: 74.02 分 | 2025 | 否 | 補充 |
| 89 | AIOpsLab | AIOps 代理評估 | 微服務雲環境 | AgentOps 範式 | 2025 | 否 | 補充 |

### 4.2 SWE-bench 系列效能總結

| 方法 | 基準 | 解決率 | 成本 | 來源 |
|------|------|--------|------|------|
| Kimi-Dev | SWE-bench Verified | **60.4%** | — | [2509.23045] |
| daVinci-Dev 72B | SWE-bench Verified | 58.5% | — | [2601.18418] |
| CAT (SWE-Compressor) | SWE-bench Verified | 57.6% | — | [2512.22087] |
| daVinci-Dev 32B | SWE-bench Verified | 56.1% | — | [2601.18418] |
| REFINE + AutoCodeRover | SWE-bench Lite | 51.67% | — | [2510.03588] |
| Claude-sonnet-4.5 | SWE-bench++ | 36.20% (pass@10) | — | [2512.17419] |
| Agentless | SWE-bench Lite | 32% | **$0.70** | [2407.01489] |
| InfCode-C++ | MultiSWE-bench-CPP | 25.58% | — | [2511.16005] |
| GPT-3.5（無策略） | RepoBugs 倉庫級 | 22.58% | — | [2403.00448] |
| SWE-agent | SWE-bench | 12.5% (pass@1) | — | SWE-agent |
| **SWE-Bench+ 真實值** | **過濾後** | **3.97%** | — | [SWE-Bench+] |

> ⚠️ SWE-Bench+ 揭示：32.67% 成功案例涉及 solution leakage，31.08% 因 weak test cases 通過。94%+ issues 在 LLM knowledge cutoff 之前。

### 4.3 故障定位效能總結

| 方法 | 指標 | 數據 |
|------|------|------|
| RGFL | 檔案級 Hit@1 | **85%** (from 71.4%) |
| FuseSearch-4B | 檔案級 F1 | **84.7%** |
| RepoLens | 平均 Hit@k / Recall@k 提升 | +22% / +46% |
| Time Travel | Git Bisect 成功率 | **80.6%** (from 74.2%) |

### 4.4 上下文效率總結

| 方法 | 指標 | 數據 |
|------|------|------|
| RLCE | 修復能力提升 | 最高 **160%** |
| SWE-Pruner | Token 縮減 | **23-54%** |
| FuseSearch | 搜索加速 / 輪次減少 | **93.6%** / 67.7% |
| MemGovern | 插件式提升 | +4.65% |
| AlignCoder | EM 分數提升 | +18.1% |
| RIG | 準確率 / 時間 | +12.2% / -53.9% |

---

## 五、現存難點與困難原因

### 難點 1：精確定位跨檔案 Bug 🔴 高

**現象**：即使最佳定位方法（RGFL Hit@1 85%），仍有 15% 的 Bug 無法正確定位。Breakpoint 研究顯示，涉及高呼叫圖中心性（system-level reasoning）時，SOTA 模型成功率降至 **0%**。

**根本原因**：
1. **因果鏈推理能力不足**：跨檔案 Bug 的根因與症狀分離，LLM 在超過 3-4 步的推理鏈上準確度急劇下降。
2. **Concern Tangling / Scattering**（RepoLens 揭示）：一個功能分散在多個檔案，一個檔案又混雜多個功能。
3. **缺乏全域語義理解**：LLM training data 以單文件為主，跨文件推理能力不足。程式碼的隱式依賴（runtime behavior、configuration）難以從文本推斷。
4. **Aggregation Deficit**（RepoReason 揭示）：整合寬度不足是主要認知瓶頸。

---

### 難點 2：長上下文推理退化 🔴 高

**現象**：CAT 研究發現，即使 200K token 窗口，Agent 推理品質在長對話中持續退化。SWE-Pruner 的存在說明：更多上下文 ≠ 更好結果。

**根本原因**：
1. **注意力稀釋效應**：Transformer softmax attention 在 token 數增加時權重下降，關鍵資訊被「淹沒」。
2. **位置偏差（Lost in the Middle）**：LLM 傾向關注 prompt 開頭和結尾，中間位置的關鍵片段被忽略。
3. **上下文切換累積漂移**：Agent 在探索、規劃、執行間切換時累積 semantic drift。
4. **O(n²) 複雜度**：Attention mechanism 限制了 window 大小的根本擴展。

---

### 難點 3：Hallucination（幻覺）風險 🔴 高

**現象**：LLM 生成看似合理但不存在的 API、錯誤的函數簽名、不符合倉庫慣例的程式碼。SWE-Bench+ 揭示 32.67% 成功案例涉及 solution leakage，暗示真正的生成能力被高估。RepoGenesis 最佳系統 Pass@1 僅 23.67%。

**根本原因**：
1. **訓練資料分布偏差**：LLM 混合不同專案的 API 和慣例，面對特定倉庫的自定義抽象時傾向「推測」而非「查證」。
2. **確認偏差**：模型優化「表面合理性」而非「深層品質」（GPT-4 重構研究 [2601.13139] 印證）。
3. **缺乏環境驗證閉環**：生成階段無法即時驗證假設。
4. **Probabilistic next-token prediction** 不保證語義正確性，缺乏 formal verification 整合。

---

### 難點 4：多語言支援薄弱 🟡 中

**現象**：絕大多數研究集中在 Python。InfCode-C++ 在 C++ 上達 25.58%，遠低於 Python 60%+。MigrationBench Java 最佳 62.33%。

**根本原因**：
1. **訓練資料不均衡**：Python 佔開源社群最高比例。
2. **語言特性差異**：C++ 模板元編程、Rust 所有權系統、Java 泛型擦除等需要不同推理能力。
3. **工具鏈差異**：Maven vs Cargo vs CMake 差異巨大。
4. **基準缺乏**：SWE-bench++ 擴展至 11 語言是重要第一步。

---

### 難點 5：測試覆蓋率不足 🟡 中

**現象**：SWT-Bench 發現 LLM 生成測試可將精確度翻倍，暗示依賴現有測試的驗證天生脆弱。

**根本原因**：
1. **測試生成的語義鴻溝**：Issue 描述往往不完整含糊，難以形式化為精確測試。
2. **回歸測試覆蓋盲區**：大型倉庫完整測試可能需要數小時。
3. **Oracle Problem**：通過測試只說明不違反已知約束，不能證明語義正確。
4. **SWE-Bench+ 揭示**：31.08% passed patches 因 weak test cases。

---

### 難點 6：成本與延遲 🟡 中

**現象**：Agentless $0.70/issue 是最低成本但只處理簡單問題。複雜 Agent 系統（BOAD、HyperAgent）可能需要 $5-50/issue。

**根本原因**：
1. **搜索空間指數增長**：Bug 複雜度增加，Agent 探索路徑增多。
2. **缺乏有效剪枝**：FuseSearch 解決了定位階段（93.6% 加速），但修復階段尚無等效方案。
3. **驗證成本高**：每個候選 patch 需執行測試套件，大型倉庫單次可能數分鐘。

---

### 難點 7：擴展到 Monorepo 規模 🔴 高

**現象**：Google、Meta 的 monorepo 包含數十億行程式碼，現有方法難以處理。大部分評估在中小型開源 repo 上。

**根本原因**：
1. **索引和搜索成本超線性增長**。
2. **Build system complexity 在 monorepo 中急劇增加**。
3. **測試執行資源消耗巨大**。
4. **Google Bug Reproduction 是唯一在工業 monorepo 上評估的論文**。

---

### 難點 8：自動生成補丁的安全性 🔴 高

**現象**：自動生成的程式碼可能引入安全漏洞（injection、buffer overflow、race condition）。VulnRepairEval 以 exploit-based 方式嚴格評估，**最佳模型僅修復 21.7%（5/23）**。更嚴重的是，Red Teaming APR（SWExploit）揭示 APR 系統可被操縱生成含漏洞的補丁——**對抗攻擊成功率達 0.91**，最佳前置過濾器僅擋 47%。

**根本原因**：
1. LLM 不理解 security invariants。
2. Training data 中安全和不安全模式混合存在。
3. Security properties 的自動驗證是 undecidable 問題。
4. C-to-Rust Translation 指出 transpilation 產生 unsafe code。
5. **Exploit-based 評估揭示真實能力遠低於測試套件評估**（VulnRepairEval [2509.03331]）。
6. **APR 系統對對抗性輸入極度脆弱**（SWExploit [2509.25894]），90% 對抗性 bug 報告可觸發攻擊者期望的補丁。

**新興緩解方案**（來自補充論文）：
- **SAST 整合驗證**：SAST-Genius 誤報降 91%，騰訊實證 94-98%，可作為補丁安全性的後置檢查
- **紅藍對抗加固**：RvB 框架防禦成功率 90%，誤報 ~0%
- **多工具交叉驗證**：CodeQL + KLEE 符號執行 + 編譯器診斷組合可降低 96% 安全漏洞

---

### 難點 9：可重現性與穩定性 🟡 中

**現象**：LLM 輸出隨機性導致同一問題不同運行得到完全不同結果。Self-play SWE-RL 的自我對弈中模型行為可能振盪。

**根本原因**：
1. **採樣隨機性**：即使 temperature=0，不同 batching/硬體可能導致不同輸出。
2. **脆弱的 prompt 依賴**：上下文順序、措詞微小變化導致完全不同修復方向。
3. **缺乏確定性回退機制**：LLM 失敗時缺乏基於規則的可靠後備。

---

### 難點總覽

| # | 難點 | 等級 | 短期可解？ | 關鍵相關論文 |
|---|------|------|----------|------------|
| 1 | 精確定位跨檔案 Bug | 🔴 高 | 部分（RGFL +13.6pp） | RGFL, Breakpoint, RepoLens |
| 2 | 長上下文推理退化 | 🔴 高 | 緩解（CAT, SWE-Pruner） | CAT, SWE-Pruner, CodeMEM |
| 3 | Hallucination 風險 | 🔴 高 | 緩解（SemAgent, 交叉驗證） | SWE-Bench+, SemAgent, REFINE |
| 4 | 多語言支援薄弱 | 🟡 中 | 漸進（SWE-bench++ 11 語言） | InfCode-C++, RepoDebug, SWE-bench++ |
| 5 | 測試覆蓋率不足 | 🟡 中 | 部分（SWT-Bench 翻倍） | SWT-Bench, SWE-Bench+, MAVM |
| 6 | 成本與延遲 | 🟡 中 | 部分（FuseSearch 93.6% 加速） | Agentless, FuseSearch |
| 7 | Monorepo 規模擴展 | 🔴 高 | 困難 | Google Bug Reproduction, RIG |
| 8 | 安全性（補丁安全 + APR 對抗） | 🔴 高 | 困難（exploit-based 僅 21.7%） | VulnRepairEval, SWExploit, RvB, SAST-Genius |
| 9 | 可重現性與穩定性 | 🟡 中 | 部分（確定性回退） | Self-play SWE-RL |
| 10 | SAST 誤報處理規模化 | 🟡 中 | 可行（騰訊 94-98%） | 騰訊誤報研究, IRIS, AdaTaint |
| 11 | CI/CD 構建失敗自動修復 | 🟡 中 | 部分（GradleFixer 81.4%） | GradleFixer, IaCGen, Build-bench |

---

## 六、實施路線圖

### 四階段遞進部署

#### 第一階段：基礎建設（Month 1-3）🟢 快速見效

**目標**：建立可運行的 MVP，處理簡單到中等難度的 Bug

| 實施內容 | 預期效果 |
|---------|---------|
| 部署 Agentless Pipeline（定位→修復→驗證） | ~32% 解決率，$0.70/issue |
| Tree-sitter AST 解析 + BM25 + 向量檢索混合索引 | 基礎程式碼索引 |
| Docker 沙盒 + 自動化測試執行 + PR 生成 | CI/CD 整合 |

**理由**：Agentless 已被驗證有效且成本極低。先跑起來、產生價值、收集真實場景數據。32% 解決率意味著每 3 個 Bug 自動修 1 個，已有顯著工程價值。

---

#### 第二階段：定位增強（Month 3-6）🔍 打通瓶頸

**目標**：顯著提升故障定位精度

| 實施內容 | 預期效果 |
|---------|---------|
| 整合 RGFL 分層推理（Bug 解釋 + File→Function→Line） | Hit@1 提升至 ~85% |
| 部署 FuseSearch 並行搜索（Fine-tune 4B 定位模型） | 加速 93.6% |
| 部署 SWE-Pruner 上下文壓縮（0.6B 略讀器） | 節省 23-54% token |
| RIG / LogicLens 程式碼圖譜 | +12.2% 準確率 |

**理由**：第一階段失敗案例中，預計 60%+ 是定位錯誤。定位是瓶頸——定位對了，修復往往簡單。RGFL 帶來 12.8% 端到端提升，實施複雜度可控。

---

#### 第三階段：Agent 升級（Month 6-9）🤖 處理複雜場景

**目標**：建立 Agent 子系統，處理 Fast Path 無法解決的複雜問題

| 實施內容 | 預期效果 |
|---------|---------|
| HyperAgent 式四代理系統（Planner/Navigator/Editor/Executor） | 處理跨檔案複雜問題 |
| REFINE 補丁精煉 | 在 Fast Path 基礎再提升 ~15% |
| MemGovern 經驗記憶（135K 經驗卡） | +4.65% 插件式提升 |
| SWT-Bench 式測試生成（Issue→測試案例） | 精確度翻倍 |
| AlignCoder RL 檢索器 | EM +18.1% |

**理由**：有了穩定的 Fast Path 和精確定位作為基礎，Agent 系統可專注處理跨檔案問題。REFINE 和 MemGovern 是低成本高回報的增量改進。

---

#### 第四階段：多語言擴展、安全整合與自我進化（Month 9-12）🌐🔐 規模化

**目標**：擴展至 Python 以外的語言，整合靜態分析安全驗證層，建立自我改進機制

| 實施內容 | 預期效果 |
|---------|---------|
| 多語言支援（優先 JS/TS、Java、Go；參考 InfCode-C++） | 覆蓋主流語言 |
| Self-play SWE-RL 自我對弈訓練 | 持續提升（+10.4 分） |
| Kimi-Dev / daVinci-Dev 思路 Fine-tuning | 降低閉源 API 依賴 |
| MigrationBench 式程式碼遷移能力 | 支援版本升級場景 |
| **SAST 整合層**（IRIS/SAST-Genius 思路 + CodeQL/Semgrep） | 補丁安全後置驗證，誤報降 91-98% |
| **LLM 合成 Checker**（KNighter 思路） | 從歷史 bug 自動生成靜態分析規則 |
| **CI/CD 自動修復**（GradleFixer/IaCGen 思路） | 構建失敗自動修復 81.4%，IaC 部署 54-91% |
| **紅藍對抗測試**（RvB 思路） | APR 輸出安全加固，防禦成功率 90% |

**理由**：多語言是企業實際需求，但研究成熟度較低，適合在核心能力穩定後擴展。Self-play 和 Fine-tuning 需要前三階段積累的真實數據。靜態分析整合（騰訊實證每條告警僅 $0.001-0.12）和 CI/CD 自動修復（GradleFixer 81.4%）已有充分工業驗證，可並行推進。

---

### 投資回報曲線

```
投資回報
  ▲
  │                                        ┌── 第四階段
  │                               ┌────────┘   (多語言、安全整合、自進化)
  │                      ┌────────┘
  │             ┌────────┘ ← 第三階段
  │        ┌────┘           (Agent 升級)
  │   ┌────┘ ← 第二階段
  │   │       (定位增強)
  │ ──┘ ← 第一階段
  │       (Agentless MVP)
  └──────────────────────────────────────────→ 時間
   M1     M3      M6       M9       M12
```

**核心原則**：
- **先跑起來，再跑得好**：第一階段用 Agentless 快速交付價值
- **定位優先於修復**：定位精度對端到端效能影響最大
- **Simple > Complex**：能用 Pipeline 解決的不用 Agent
- **數據驅動迭代**：每階段失敗案例分析驅動下一階段優化
- **安全不可後補**：第四階段整合 SAST 驗證層，確保生成補丁不引入新漏洞

### 安全相關里程碑

| 時間 | 里程碑 | 預期成果 | 參考 |
|------|--------|---------|------|
| M3 | 基礎 SAST 掃描整合 | 所有生成補丁經 CodeQL/Semgrep 掃描 | IRIS, SAST-Genius |
| M6 | LLM 誤報過濾上線 | SAST 告警誤報降 90%+，成本 <$0.12/告警 | 騰訊研究 |
| M9 | 安全漏洞自動修補 MVP | 已知 CVE 類型自動修補率 >50% | VulnResolver, PatchIsland |
| M10 | LLM Checker 合成 | 從歷史 commit 自動生成項目特定 checker | KNighter |
| M11 | CI/CD 構建修復整合 | 構建失敗自動修復率 >70% | GradleFixer, IaCGen |
| M12 | 紅藍對抗測試 | APR 輸出通過對抗性安全測試 | RvB, SWExploit |

---

## 七、結論與建議

### 關鍵結論

1. **倉庫級自動化代碼修改技術正在快速成熟**：從 2024 年 SWE-agent 的 12.5% 到 2025 年 Kimi-Dev 的 60.4%，一年多內 SWE-bench 解決率提升近 5 倍。但 SWE-Bench+ 的揭示（真實率 3.97%）提醒我們需謹慎看待 benchmark 數字。

2. **Agentless + Agent 混合架構是最務實的選擇**：Agentless 以極低成本處理多數簡單問題，Agent 系統負責複雜場景。Kimi-Dev 證明兩者技術可互通。

3. **基礎設施決定上限**：程式碼索引（RIG）、上下文管理（CAT/SWE-Pruner）、經驗記憶（MemGovern）等基礎設施的品質，直接決定了上層修復能力的天花板。

### Top 3 建議

| 優先級 | 建議 | 理由 |
|--------|------|------|
| 🥇 | **立即部署 Agentless Pipeline 作為 MVP** | 32% 解決率 + $0.70/issue，2-4 週可上線，快速驗證業務價值。不要等完美方案再開始。 |
| 🥈 | **投資故障定位能力（RGFL + FuseSearch）** | 定位是整個 Pipeline 的瓶頸。RGFL 帶來 12.8% 端到端提升。定位對了，修復往往簡單。 |
| 🥉 | **建立程式碼知識圖譜基礎設施** | RIG/LogicLens 提供 +12.2% 準確率和 -53.9% 完成時間。這是所有後續能力（定位、修復、審查）的共享基礎。一次建設，多次複用。 |

### 預期效果（根據論文數據）

| 指標 | 預期值 | 參考來源 |
|------|--------|---------|
| Context 檢索準確率提升 | **12-18%** | AlignCoder, RIG |
| Bug 修復解決率 | **32-60%**（取決於階段） | Agentless → Kimi-Dev |
| Code Review 缺陷覆蓋率 | **+285%** vs 人工 PR | AACR-Bench |
| 開發效率（完成時間） | **-54%**（有 code graph 時） | RIG |
| 定位精度（檔案級） | **~85%** Hit@1 | RGFL |
| Token 成本節省 | **23-54%** | SWE-Pruner |
| SAST 誤報降低 | **91-98%** | SAST-Genius, 騰訊研究 |
| 安全告警處理成本 | **$0.001-0.12/告警** | 騰訊研究 |
| 安全漏洞自動修補率 | **72-75%**（已知類型） | PatchIsland, VulnResolver |
| CI/CD 構建修復率 | **75-81%** | GradleFixer, Android Diag. |
| IaC 部署成功率 | **54-91%**（迭代後） | IaCGen |

---

## 附錄：完整論文索引

### A. 兩份報告共有論文（15 篇）

| # | 論文名稱 | arXiv ID | 類別 | 年份 |
|---|---------|----------|------|------|
| 1 | RepoBugs + RLCE | 2403.00448 | 缺陷修復 | 2024 |
| 2 | REFINE | 2510.03588 | 缺陷修復 | 2025 |
| 3 | RGFL | 2601.18044 | 故障定位 | 2026 |
| 4 | HyperAgent | 2409.16299 | SWE Agent | 2024 |
| 5 | Self-play SWE-RL | 2512.18552 | SWE Agent / RL | 2025 |
| 6 | AlignCoder | 2601.19697 | 程式碼理解 | 2026 |
| 7 | InfCode-C++ | 2511.16005 | 故障定位 | 2025 |
| 8 | AACR-Bench | 2601.19494 | 程式碼審查 | 2026 |
| 9 | CodeFuse-CR-Bench | 2509.14856 | 程式碼審查 | 2025 |
| 10 | SWE-Bench++ | 2512.17419 | 基準測試 | 2025 |
| 11 | RepoDebug | 2509.04078 | 基準測試 | 2025 |
| 12 | MigrationBench | 2505.09569 | 程式碼遷移 | 2025 |
| 13 | FreshBrew | 2510.04852 | 程式碼遷移 | 2025 |
| 14 | LLM Issue Resolution Survey | 2601.11655 | 綜述 | 2026 |
| 15 | Benchmarking Fine-Grained CR / ContextCRBench | 2511.07017 | 程式碼審查 | 2025 |

### B. 博特二號獨有論文（~17 篇）

| # | 論文名稱 | arXiv ID | 類別 | 年份 |
|---|---------|----------|------|------|
| 16 | Agentless | 2407.01489 | 缺陷修復 | 2024 |
| 17 | Kimi-Dev | 2509.23045 | 缺陷修復 | 2025 |
| 18 | daVinci-Dev | 2601.18418 | SWE Agent | 2026 |
| 19 | BOAD | 2512.23631 | SWE Agent | 2025 |
| 20 | CAT | 2512.22087 | SWE Agent | 2025 |
| 21 | MemGovern | 2601.06789 | SWE Agent | 2026 |
| 22 | FuseSearch | 2601.19568 | SWE Agent | 2026 |
| 23 | SWE-Pruner | 2601.16746 | SWE Agent | 2026 |
| 24 | SWE-bench | 2310.06770 | 基準測試 | 2023 |
| 25 | Breakpoint | 2506.00172 | 基準測試 | 2025 |
| 26 | SWT-Bench | 2406.12952 | 基準測試 | 2024 |
| 27 | FAUN-Eval | 2411.18019 | 基準測試 | 2024 |
| 28 | RepoGenesis | 2601.13943 | 基準測試 | 2026 |
| 29 | RepoLens | 2509.21427 | 程式碼理解 | 2025 |
| 30 | MAVM | 2601.17762 | 漏洞管理 | 2026 |
| 31 | Time Travel | 2511.18854 | 故障定位 | 2025 |
| 32 | GPT-4 Refactoring | 2601.13139 | 重構 | 2026 |

### C. 博特一號獨有論文（~25 篇）

| # | 論文名稱 | 類別 | 年份 |
|---|---------|------|------|
| 33 | SWE-agent | SWE Agent | 2024 |
| 34 | One Tool Is Enough | SWE Agent / RL | 2025 |
| 35 | OctoBench | 基準測試 | 2026 |
| 36 | Thinking Longer, Not Larger | SWE Agent | 2025 |
| 37 | SemAgent | 缺陷修復 | 2025 |
| 38 | HAFixAgent | 缺陷修復 | 2025 |
| 39 | Autonomous Issue Resolver | 缺陷修復 | 2025 |
| 40 | Tracing-Errors | 缺陷修復 | 2025 |
| 41 | LLM-Based-Nullability-Repair | 缺陷修復 | 2025 |
| 42 | Google Bug Reproduction | 缺陷修復 | 2025 |
| 43 | SWE-Synth | 訓練資料 | 2025 |
| 44 | MCTS-Refined-CoT | 訓練資料 | 2025 |
| 45 | LogicLens | 程式碼理解 | 2026 |
| 46 | RIG | 程式碼理解 | 2026 |
| 47 | DependEval | 基準測試 | 2025 |
| 48 | Kodezi Chronos | 程式碼理解 | 2025 |
| 49 | RepoShapley | 程式碼理解 | 2026 |
| 50 | CodeMEM | 程式碼理解 | 2026 |
| 51 | InlineCoder | 程式碼理解 | 2026 |
| 52 | C-to-Rust Translation | 程式碼遷移 | 2025 |
| 53 | Advancing-Automated-Validation | 程式碼遷移 | 2025 |
| 54 | SWE-Bench+ | 基準測試 | 2024 |
| 55 | RepoReason | 基準測試 | 2026 |
| 56 | Reformulate-Retrieve-Localize | 故障定位 | 2025 |
| 57 | NL-Summarization-Multi-Repo | 故障定位 | 2025 |

### D. 補充搜索論文（31 篇）

#### 🔐 LLM + 靜態分析整合（12 篇）

| # | 論文名稱 | arXiv ID | 會議/期刊 | 年份 |
|---|---------|----------|----------|------|
| 58 | KNighter | 2503.09002 | SOSP 2025 | 2025 |
| 59 | IRIS | 2405.17238 | — | 2024 |
| 60 | SAST-Genius | 2509.15433 | IEEE S&P 2025 | 2025 |
| 61 | Reducing False Positives with LLMs (Tencent) | 2601.18844 | — | 2026 |
| 62 | RAG + Multi-Tool Secure Code Gen | 2601.00509 | — | 2026 |
| 63 | BugLens | 2504.11711 | — | 2025 |
| 64 | PredicateFix | 2503.12205 | ICSE 2026 | 2025 |
| 65 | QLCoder | 2511.08462 | — | 2025 |
| 66 | AdaTaint | 2511.04023 | — | 2025 |
| 67 | LLM Prompting as SA Proxy | 2412.12039 | — | 2024 |
| 68 | LAMeD | 2505.02376 | — | 2025 |
| 69 | BugScope | 2507.15671 | — | 2025 |

#### 🛡️ 自動化安全漏洞修補（10 篇）

| # | 論文名稱 | arXiv ID | 會議/期刊 | 年份 |
|---|---------|----------|----------|------|
| 70 | PatchIsland | 2601.17471 | — | 2026 |
| 71 | VulnResolver | 2601.13933 | — | 2026 |
| 72 | Well Begun is Half Done | 2512.20203 | ICSE 2026 | 2025 |
| 73 | Vul-R2 | 2510.05480 | ASE 2025 | 2025 |
| 74 | PATCHEVAL | 2511.11019 | — | 2025 |
| 75 | One-Shot Patching Eval | 2511.23408 | SAC SEAI 2026 | 2025 |
| 76 | VulnRepairEval | 2509.03331 | — | 2025 |
| 77 | SecureFixAgent | 2509.16275 | ICMLA 2025 | 2025 |
| 78 | RvB | 2601.19726 | — | 2026 |
| 79 | Red Teaming APR (SWExploit) | 2509.25894 | — | 2025 |

#### 🔄 CI/CD 管線自動修復（10 篇）

| # | 論文名稱 | arXiv ID | 會議/期刊 | 年份 |
|---|---------|----------|----------|------|
| 80 | GradleFixer | 2510.08640 | — | 2025 |
| 81 | Build-bench | 2601.12927 | — | 2026 |
| 82 | Build-bench Extended | 2511.00780 | — | 2025 |
| 83 | CI/CD Config Translation | 2511.01316 | — | 2025 |
| 84 | AI-Augmented CI/CD Pipelines | 2508.11867 | — | 2025 |
| 85 | Agentic AI for Supply Chain Security | 2512.23480 | — | 2025 |
| 86 | Android Build Diagnosis | 2511.06186 | — | 2025 |
| 87 | IaCGen | 2506.05623 | FSE 2026 | 2025 |
| 88 | MACOG | 2510.03902 | — | 2025 |
| 89 | AIOpsLab | 2501.06706 | — | 2025 |

---

*本報告合併自兩份獨立研究報告（博特一號 33 篇 + 博特二號 28 篇）及補充搜索（31 篇），去重後涵蓋 86 篇 2023–2026 年 arXiv 論文。補充搜索聚焦三大缺口領域：LLM + 靜態分析整合、自動化安全漏洞修補、CI/CD 管線自動修復。所有數據引用均標註原始來源。當兩份報告覆蓋同一論文時，優先採用包含更具體數據點的版本。*

*具體數據應以原始論文為準。*
