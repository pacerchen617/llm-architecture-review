# 多 Agent 系統架構全景 — 2026-04-07

> 本文件供外部 reviewer 審查我們的 AI agent 架構設計。
> 涵蓋：Agent 分工、LLM 算力池、記憶系統、知識圖譜（Obsidian）、自動化 pipeline、跨 agent 通訊。
> 最後更新：2026-04-07 10:00 GMT+8

---

## 目錄

0. [背景與問題定義](#0-背景與問題定義)
1. [Agent 架構](#1-agent-架構)
2. [四層算力池（LLM Compute Pool）](#2-四層算力池llm-compute-pool)
3. [品質閘門 v2（三層）](#3-品質閘門-v2三層)
4. [記憶系統 v2.0](#4-記憶系統-v20)
5. [Obsidian 知識圖譜](#5-obsidian-知識圖譜)
6. [自動化 Pipeline（67 個 launchd 排程）](#6-自動化-pipeline67-個-launchd-排程)
7. [跨 Agent 通訊](#7-跨-agent-通訊)
8. [Instinct Pipeline + 自我進化](#8-instinct-pipeline--自我進化)
9. [API 用法參考](#9-api-用法參考)
10. [併發安全 + 版本釘選](#10-併發安全--版本釘選)
11. [已知問題 + 技術債](#11-已知問題--技術債)
12. [技術選型淘汰條件](#12-技術選型淘汰條件)
13. [檔案索引](#13-檔案索引)

---

## 0. 背景與問題定義

### 硬體

一台 16GB MacBook（M3），跑 5 個 AI agent + 本地 LLM + 2 個 gateway + 排程系統。

### 核心約束

**$0 增量成本**。所有 LLM 都走已付費訂閱額度或免費 API：
- Claude Max 訂閱 → Opus 4.6 + Sonnet 4.6（CLIProxyAPI 本地代理）
- ChatGPT Plus 訂閱 → GPT-5.4（OpenClaw gateway 代理）
- Google AI Free → Gemini 2.5 Flash
- Ollama 本地 → qwen3.5:9b + qwen3-embedding:0.6b

### 為什麼需要地端 LLM

| 問題 | 雲端限制 | 地端解法 |
|------|---------|---------|
| 延遲 | 3-5 秒/次，pipeline 每天跑數十次 | ~200ms，離線可用 |
| Token 消耗 | 排程任務佔主 agent context window | 獨立運算，不消耗雲端額度 |
| 穩定性 | 網路斷 = pipeline 停 | 完全離線運作 |

---

## 1. Agent 架構

### 活躍 Agent（5 個）

| Agent | 框架 | 模型 | 角色 | TG Bot |
|-------|------|------|------|--------|
| 🏭 夢工廠 | Claude Code (local CLI) | Opus 4.6 | CTO — 程式開發、系統管理、Skill 建置 | @PacerCraw_bot |
| 📊 投放者 | Claude Code (local CLI) | Opus 4.6 | Meta 廣告管理、CPA 優化 | @PacerFbad_bot |
| 🧵 社群飛輪 | OpenClaw gateway | GPT-5.4 | Threads 社群內容自動化 | @PacerThread01_bot |
| 🧗 體能工作室 | OpenClaw gateway | GPT-5.4 | 耐力運動教練 | @DrFrank617_bot |
| 🔧 101 Engineer | OpenClaw gateway | Opus 4.6 | 前端/GTM/GCP 部署 | — |

### 已退休

| Agent | 退休日 | 原因 |
|-------|--------|------|
| 🦞 蝦蝦 | 2026-04-05 | Anthropic OAuth 終止，COO/PM 職能分散到各 agent |

### 兩種框架差異

```
Claude Code（夢工廠、投放者）          OpenClaw（社群飛輪、體能工作室、101）
┌─────────────────────────┐        ┌──────────────────────────────┐
│ 本地 CLI 直接跑          │        │ OpenClaw Gateway (port 18789) │
│ 可用所有 Mac 工具         │        │ → CLIProxyAPI → Claude/GPT    │
│ 完整 Skill 存取           │        │ 有限工具集（TG + 記憶）         │
│ 記憶：~/.claude-bot-memory│        │ 記憶：workspace-*/memory/      │
│ 觸發：TG + cron + signal  │        │ 觸發：TG + cron + signal       │
└─────────────────────────┘        └──────────────────────────────┘
```

### 架構特性：異質 Agent

這不是同一個 LLM 的多個 instance。每個 agent 是**獨立個體**：
- 不同框架（Claude Code vs OpenClaw）
- 不同模型（Opus vs GPT-5.4）
- 不同記憶空間
- 不同人格和溝通風格
- 共享：Mac 檔案系統、Skill 庫、Obsidian Vault

---

## 2. 四層算力池（LLM Compute Pool）

```
Layer 0 即時層  │ qwen3.5:9b 本地 ($0, ~200ms, 離線, 無限量)
               │ → 分類/路由/初篩/快速生成
               │
Layer 1 品質層  │ Sonnet 4.6 via CLIProxyAPI ($0 Claude Max)
               │ → 品質閘門 second opinion / 高頻摘要
               │
Layer 2 深度層  │ Opus 4.6 via CLIProxyAPI ($0 Claude Max)
               │ → 知識聚合 / 記憶衝突裁決 / 複雜推理
               │
Layer 3 獨家層  │ Gemini Flash ($0 Free, Google Search grounding)
               │ + GPT-5.4 ($0 ChatGPT Plus, 跨架構驗證)
```

### 路由表（ModelDispatcher）

| 任務類型 | 主力 | 備援 | 選擇邏輯 |
|---------|------|------|---------|
| 品質判斷 (judge) | Sonnet 4.6 | Opus 4.6 | 不同模型=真 second opinion |
| 深度推理 (reason) | Opus 4.6 | Sonnet 4.6 | 最強推理做最難的事 |
| 即時分類 (classify) | qwen3.5:9b | Sonnet 4.6 | 本地 0ms，掛了才走雲端 |
| 摘要 (summarize) | Sonnet 4.6 | qwen3.5:9b | 品質重要 |
| 研究 (research) | Gemini + grounding | Sonnet 4.6 | 唯一有 Google Search |
| 知識聚合 (aggregate) | Opus 4.6 | Sonnet 4.6 | 1M context + 128K output |
| 跨架構驗證 (validate) | GPT-5.4 | Opus 4.6 | 不同架構=不同盲點 |
| Embedding (embed) | qwen3-embed:0.6b | — | 唯一選擇 |
| 一般生成 (generate) | qwen3.5:9b | Sonnet 4.6 | 本地優先 |

### 資源消耗（16GB Mac）

| 狀態 | RAM |
|------|-----|
| Idle（只有 embedding 常駐） | ~1.4 GB |
| 推理中（qwen3.5 + embedding） | ~7.2 GB |
| 系統剩餘 | ~8.8 GB |

---

## 3. 品質閘門 v2（三層）

```
輸入（新 lesson / 新 instinct / 內容品質檢查）
  ↓
Layer 1: Heuristic Noise Filter（0ms，純正則+規則）
  ├─ score < 0.3 → 直接 REJECT
  ├─ score 0.3-0.6 → 灰色地帶 → Layer 2
  └─ score > 0.6 → 直接 PASS
  ↓
Layer 2: qwen3.5:9b 本地初篩（~200ms）
  ├─ score ≤ 2 → REJECT
  ├─ score = 3 → 灰色地帶 → Layer 3
  └─ score ≥ 4 → PASS
  ↓
Layer 3: Sonnet 4.6 Second Opinion（灰色地帶才觸發）
  ├─ 取兩者較低分（保守策略）
  ├─ 兩者一致 → 直接採用
  └─ Sonnet 不可用 → 回退用 L2 分數
```

**設計哲學**：確定性驗證優先。Heuristic 成本 0ms 先篩掉 70% 的噪音，LLM 只處理灰色地帶。

---

## 4. 記憶系統 v2.0

### 設計原則

```
記結論不記過程 → 高密度記憶
MEMORY.md 是目錄不是百科 → 指向 truth source
每種事實只有一個權威來源 → truth-sources.json
記憶可能過時 → 使用前先驗證
```

### 記憶分層

```
┌─────────────────────────────────────────────────────┐
│ Constitution（不可自改）                               │
│   SOUL.md, CLAUDE.md, IDENTITY.md                    │
├─────────────────────────────────────────────────────┤
│ Governed（可提案，Kai 批准）                           │
│   strategies.json, HEARTBEAT.md, USER.md             │
├─────────────────────────────────────────────────────┤
│ Adaptive（pipeline 自動更新，有護欄）                   │
│   learning-log.jsonl, lessons.md, MEMORY.md          │
│   hypotheses.jsonl, heuristics.md                    │
└─────────────────────────────────────────────────────┘
```

### 記憶生命週期

```
事件發生
  ↓
learning-log.jsonl（append-only，每條記錄一個結論）
  ↓ cognize（去重 + NLI 衝突偵測）
  ↓ link（token Jaccard ≥ 0.45 → 建立交叉連結）
  ↓ decay（每日 03:00，低 confidence 衰減）
  ↓ consolidate（每週日，合併相似記憶）
  ↓ graduate（通過品質閘門 → Obsidian Vault/Lessons/）
```

### 消化引擎 v10

工具：`learning-log-utils.py`，支援操作：

| 操作 | 作用 |
|------|------|
| `cognize` | 寫入新記憶（自動分類 ADD/UPDATE/SKIP/CONFLICT） |
| `link` | 建立記憶間的交叉連結（A-MEM graph） |
| `recall` | 按相關度召回記憶 |
| `decay` | 衰減低 confidence 記憶 |
| `consolidate` | 合併重複記憶 |
| `graduate` | 升格到 Obsidian 知識圖譜 |

### 記憶類型（MIRIX 分類，0ms heuristic）

| 類型 | 範例 |
|------|------|
| `episodic` | 「04/06 修了 GTM import bug」 |
| `semantic` | 「CJK token 比 Latin 多 2-3x」 |
| `procedural` | 「部署到 GCS 的步驟是…」 |
| `core` | 「$0 增量成本是核心約束」 |

### 跨 Session 延續

```
Session N 結束
  ↓ PreCompact hook
session-tail.md（最後 ~15K context 自動保存）
  ↓
Session N+1 開始
  ↓ 讀 session-tail.md + active-tasks.json
接續上次工作
```

---

## 5. Obsidian 知識圖譜

### 為什麼 Obsidian 是第一公民

```
Agent 的 learning-log → 短期記憶（天為單位）
Agent 的 lessons.md   → 中期索引（週為單位）
Obsidian Vault        → 長期知識本體（永久，可搜尋，有結構）
```

Obsidian 不只是筆記工具，它是 agent 的**第二大腦**。所有經過品質閘門的知識最終都 graduate 到 Vault。

### Vault 結構（21 個資料夾，~450+ 檔案）

| 資料夾 | 檔案數 | 用途 |
|--------|--------|------|
| `Lessons/` | 116 | 按領域分類的經驗教訓（知識圖譜核心） |
| `Skills/` | 60 | 每個 Skill 的文件（scope、trigger、verification） |
| `Decisions/` | 58 | ADR（Architecture Decision Record） |
| `Services/` | 56 | 每個服務的文件（config、health、dependency） |
| `Changelog/` | 38 | 系統變更紀錄 |
| `Memory/` | 35 | 記憶系統相關文件 |
| `Learnings/` | 30 | 每日學習摘要（daily digest） |
| `Reports/` | 27 | 自動生成的報告 |
| `Architecture/` | 27 | 系統設計文件 |
| `Templates/` | 11 | 標準化模板（筆記、決策、服務） |

### 知識流入 Obsidian 的三條路

```
路徑 1: 自動畢業
  learning-log.jsonl → quality_gate ≥ 4 → graduate → Vault/Lessons/

路徑 2: 決策紀錄
  S/A 級任務完成 → ADR 寫入 Vault/Decisions/

路徑 3: L2 知識聚合（週報）
  每週日 Opus 4.6 聚合一週 learning-log → Vault/Learnings/weekly-trend-*.md
```

### 知識搜尋（QMD Hybrid Search）

```
查詢
  ↓
QMD Router
  ├─ BM25 快速（~180ms）→ 精確關鍵字匹配
  └─ Vector + RRF reranker（~30s）→ 語意搜尋
  ↓
結果（含 Vault 路徑 + 相關度分數）
```

- Embedding model: `qwen3-embedding:0.6b`（常駐記憶體，keep_alive=-1）
- 索引更新: `qmd collection add <path> --update && qmd embed`

### 知識圖譜的「成長」問題

**現狀**：Vault 有 450+ 檔案，但「自我成長」機制存在瓶頸：

| 階段 | 機制 | 狀態 |
|------|------|------|
| 知識寫入 | graduate pipeline | ✅ 運作中，每日 ~3-5 條 |
| 知識連結 | A-MEM graph linking | ✅ Jaccard ≥ 0.45 自動連結 |
| 知識搜尋 | QMD hybrid search | ✅ BM25 + Vector |
| 知識清理 | vault-lint.sh (03:15) | ⚠️ 首次發現 42 頁漂移 |
| 知識合成 | weekly-trend L2 | ✅ 每週 Opus 聚合 |
| 知識淘汰 | decay pipeline | ⚠️ 只作用於 learning-log，未觸及 Vault 過時筆記 |
| 主動知識缺口偵測 | curiosity-loop (04:30) | ✅ 追蹤 interest-queue → 自動研究 |

**待改善**：
1. Vault 內過時筆記無自動淘汰/標記機制（只有 learning-log 有 decay）
2. Vault 筆記之間的 wiki-link 大多靠人工或 graduate 時產生，缺少批次回補
3. `Lessons/` 116 個檔案部分重複，需要 consolidation 機制
4. `Architecture/System Overview.md` 仍描述雙 agent 時代，過時

---

## 6. 自動化 Pipeline（67 個 launchd 排程）

### 核心服務（Always-On）

| 服務 | Port | 作用 |
|------|------|------|
| OpenClaw Gateway | 18789 | GPT-5.4 agent 的 API 代理 |
| CLIProxyAPI | 8317 | Claude Max 訂閱的本地代理 |
| 夢工廠 TG Bot | — | grammY → Claude Code |
| Signal Watcher | — | `signals/queue.jsonl` → agent-relay.sh v5 |
| Factory Inbox | — | `signals/factory-inbox.jsonl` → inbox-watcher.ts |

### 每日排程（Nightly Pipeline）

```
03:00  daily-backup
       ├─ chezmoi snapshot → GCS
       ├─ Vault sync → GCS
       ├─ instinct-engine digest --use-llm
       └─ audit trail

03:15  vault-health-report（lint 42 項）

03:30  changelog-digest → TG 通知

04:00  witness-process（備份見證）

04:30  curiosity-loop → interest-queue → learning-inbox/

05:00  inbox-digest + ClimbLab sync

05:05  insight-router（學習日報分析 → proposals）
```

### 社群飛輪 Pipeline

```
07:00  trend-scanner → 趨勢抓取
08:00  research-consolidated → 深度研究
08:15  research-round2 → 二次驗證
09:00  research-round3 → 三次交叉
10:30  daily-briefing → 簡報生成
11:00  recommender → 選題推薦
       → daily-planner → 排程規劃
       → publisher → 自動發布（有 Gate 攔截）
22:00  post-verify → 發文後驗證
```

### Meta 廣告 Pipeline

```
每日   daily-campaign-report → CPA 報告
每週   weekly-eval → 效能評估
每月   monthly-check → 趨勢檢查
持續   patrol-watchdog → 異常監控
```

### 記憶維護 Pipeline

```
每日 03:00  instinct digest + decay
每日 05:05  insight-router → proposals
每週日      consolidate + graduate + weekly-trend L2
```

---

## 7. 跨 Agent 通訊

### 通訊架構

```
┌─────────┐  signal    ┌──────────────┐  relay   ┌──────────┐
│ Agent A │──────────→│ queue.jsonl  │────────→│ Agent B  │
└─────────┘           └──────────────┘          └──────────┘
                            ↓
                     agent-relay.sh v5
                     (讀取 → 路由 → 投遞)

┌─────────────────────────────────────┐
│ TG Shared Group: 指揮中心             │
│  ├─ 🔧 維護 (topic 552)              │
│  ├─ 🔬 研究 (topic 875)              │
│  └─ 🌐 落地頁 (topic 8586)           │
│  夢工廠 Forum Group（獨立）            │
└─────────────────────────────────────┘
```

### 通訊方式比較

| 方式 | 延遲 | 用途 |
|------|------|------|
| Signal Queue (`queue.jsonl`) | 秒級 | Agent 間結構化通知 |
| Factory Inbox (`factory-inbox.jsonl`) | 秒級 | 其他 agent → 夢工廠的請求 |
| Agent Mailbox (`~/.openclaw/mailboxes/`) | 分鐘級 | 非即時結構化訊息 |
| TG Shared Group | 即時 | 人類可見的 agent 對話 |

### 共享資源

| 資源 | 路徑 | 共享範圍 |
|------|------|---------|
| Skill 庫 | `~/.claude/skills/` | 夢工廠 + 投放者（Claude Code agents） |
| `_lib/models.py` | `~/.claude/skills/_lib/` | 所有使用地端 LLM 的 agent |
| Obsidian Vault | `~/Documents/Obsidian Vault/` | 所有 agent（讀寫） |
| Agent Registry | `~/.openclaw/agent-registry.json` | 所有 agent（唯讀通訊錄） |
| Truth Sources | `~/.claude-bot-memory/truth-sources.json` | 夢工廠主控 |

---

## 8. Instinct Pipeline + 自我進化

### Instinct 生命週期

```
PostToolUse hook → auto-observe.sh → reflect-queue.jsonl
                                         ↓
daily-backup 03:00 → instinct-engine digest --use-llm
                                         ↓
                     qwen3.5:9b 分類 + 摘要 + quality_gate
                                         ↓ (score ≥ 3)
                     instincts.jsonl → evolve (weekly)
                                         ↓ (confidence ≥ 0.65)
                     learning-log.jsonl → lessons.md → Obsidian Vault
```

### 進化基因（Constitution 層）

10 條不可自改的基因，核心：
1. 遇到新架構模式 → 先研究再判斷
2. 完成任務 → 自問「同原則適用其他 agent / 檔案嗎？」
3. 每次架構決策 → 記錄到 Obsidian 決策圖譜
4. 所有技術選型都是**暫時最優解** — 附帶自己的淘汰條件
5. 系統健康 > 功能完整 — 穩定優先

### 好奇心驅動學習

```
工作中遇到知識缺口
  ↓
interest-queue.jsonl（追蹤主題）
  ↓
curiosity-loop.py（每日 04:30 自動研究）
  ↓
learning-inbox/ → 消化 → learning-log
```

---

## 9. API 用法參考

### 統一 API：`_lib/models.py`

```python
from _lib.models import (
    LLMResult,          # 統一回傳類型
    ModelDispatcher,    # 任務導向路由
    dispatcher,         # singleton instance
    call_model,         # Claude via CLIProxyAPI → LLMResult
    call_ollama,        # Local Ollama → string（向後兼容）
    call_ollama_result, # Local Ollama → LLMResult
    call_gemini,        # Gemini API → LLMResult
    call_codex,         # GPT via Codex CLI → LLMResult
    quality_gate,       # 三層品質閘門 → dict
)
```

### LLMResult

```python
@dataclass
class LLMResult:
    success: bool
    content: str
    error: str | None = None
    model: str = ""
    provider: str = ""
    tokens_used: int = 0
    latency_ms: int = 0

result = call_model("claude-sonnet-4-6", "system prompt", "user input")
if result.success:
    print(result.content)
```

### ModelDispatcher — 按任務自動路由

```python
result = dispatcher.call("評估這段輸出", task_type="judge")     # → Sonnet
result = dispatcher.call("搜尋最新...", task_type="research")    # → Gemini
result = dispatcher.call("分析這個架構", task_type="reason")     # → Opus
# 主力失敗 → 自動切備援
```

### quality_gate v2

```python
qg = quality_gate(output="...", task="...")
# {"score": 4, "reason": "...", "pass": True, "judge": "qwen3.5:9b+sonnet-4.6"}
```

---

## 10. 併發安全 + 版本釘選

### Ollama fcntl.flock

```python
# call_ollama_result() 內建 file lock
# 路徑：/tmp/ollama-call.lock
# 效果：序列化所有 Ollama 呼叫，防止 16GB OOM
```

### 模型版本釘選

```json
// ~/.claude-bot-memory/model-versions.json
{
  "pinned": true,
  "models": {
    "qwen3.5:9b": {
      "digest": "6488c96fa5fa",
      "pinned_reason": "Round 2 Benchmark 通過",
      "unpinned_condition": "新版 benchmark 通過且品質 >= 4.4"
    }
  }
}
```

`ollama-update.sh`（每週日）檢查 digest，pinned 模型不會被自動更新。

---

## 11. 已知問題 + 技術債

### 架構層

| 問題 | 影響 | 優先級 |
|------|------|--------|
| 蝦蝦退休後 PM 角色空缺 | 指揮中心 topic 2（任務分派）無 agent 負責 | 🟡 中 |
| `social-signals` agent 未完成 | registry 有 placeholder，無實際功能 | 🟢 低 |
| OpenClaw OAuth 遷移進行中 | Phase 2，部分 agent 仍依賴舊認證 | 🔴 高 |

### 記憶 / 知識層

| 問題 | 影響 | 優先級 |
|------|------|--------|
| Obsidian Vault 筆記無自動淘汰 | 過時筆記累積，搜尋結果品質下降 | 🟡 中 |
| `Architecture/System Overview.md` 過時 | 仍描述雙 agent 架構 | 🟡 中 |
| `Lessons/` 116 檔部分重複 | 需要 consolidation | 🟢 低 |
| `strategies.json` 結構可能空/損壞 | instinct engine 無策略知識可用 | 🟡 中 |

### 服務 / 自動化層

| 問題 | 影響 | 優先級 |
|------|------|--------|
| `running-services.md` 只記錄 28/67 個 plist | 文件嚴重落後現況 | 🟡 中 |
| 蝦蝦遺留 11 個 skill 無 owner | skill-catalog 仍指向已退休 agent | 🟡 中 |
| `gemini-chat`、`identity-lint` 未列入 catalog | skill 存在但無文件 | 🟢 低 |
| 67 個 plist 中部分功能重疊 | 維護成本增加 | 🟢 低 |

---

## 12. 技術選型淘汰條件

| 選擇 | 為什麼 | 淘汰條件 |
|------|--------|---------|
| Sonnet 4.6 品質閘門 | 高頻+品質夠強 | Claude Max 取消訂閱 |
| Opus 4.6 L2 聚合 | 最強推理+128K output | 出現更強且 $0 替代 |
| qwen3.5:9b 本地 | Round 2 驗證+省 RAM | Mac Mini M4 Pro 64GB → 升級 26B+ |
| Gemini Flash grounding | Google Search 獨家 | Claude 原生搜尋開放 |
| CLIProxyAPI | $0 Claude Max 本地代理 | Anthropic 封鎖本地代理 |
| OpenClaw gateway | GPT-5.4 代理 | OpenAI 開放原生 API 免費層 |
| fcntl.flock | 單機簡單有效 | 多機部署 → 分散式鎖 |
| JSONL 記憶 | 簡單+chezmoi 友好 | >5000 條 → SQLite |
| Obsidian + QMD | Markdown + 語意搜尋 | Vault >2000 檔 → 考慮專用知識庫 |
| launchd plist | macOS 原生排程 | 跨平台需求 → 改 systemd/k8s cron |

---

## 13. 檔案索引

### 核心程式

| 用途 | 路徑 |
|------|------|
| LLM API（v2） | `~/.claude/skills/_lib/models.py` |
| Instinct Engine | `~/.local/bin/instinct-engine.py` |
| Memory Utils v10 | `~/.local/bin/learning-log-utils.py` |
| L2 Trend Analysis | `~/.local/bin/weekly-trend-l2.py` |
| Weekly Compound | `~/.local/bin/memory-weekly-compound.sh` |
| Ollama Update | `~/.local/bin/ollama-update.sh` |
| Agent Relay v5 | `~/.local/bin/agent-relay.sh` |
| Vault Sync | `~/.local/bin/sync-vault.sh` |
| Vault Lint | `~/.local/bin/vault-lint.sh` |
| Curiosity Loop | `~/.local/bin/curiosity-loop.py` |
| Insight Router | `~/.local/bin/insight-router.py` |

### 設定檔

| 用途 | 路徑 |
|------|------|
| Agent Registry | `~/.openclaw/agent-registry.json` |
| Model Versions | `~/.claude-bot-memory/model-versions.json` |
| Truth Sources | `~/.claude-bot-memory/truth-sources.json` |
| Active Tasks | `~/.claude-bot-memory/active-tasks.json` |
| Agent Mailbox | `~/.openclaw/mailboxes/` |
| Skill Catalog | `~/.openclaw/workspace/skill-catalog.md` |

### Obsidian Vault

| 資料夾 | 內容 |
|--------|------|
| `Architecture/` | 系統設計文件（LLM 架構、記憶系統、品質閘門） |
| `Decisions/` | ADR（架構決策紀錄） |
| `Lessons/` | 經驗教訓知識圖譜 |
| `Skills/` | Skill 文件 |
| `Services/` | 服務文件 |
| `Learnings/` | 每日/每週學習摘要 |
| `Templates/` | 標準化模板 |

---

## 附錄：決策歷程

### Round 1 → Round 2 Benchmark

Qwen3.5 9B 在公平 Round 2 測試中勝出 — 速度快 22%、省 31% RAM、繁中品質 4.48/5。

### 2026-04-06 四層算力池升級

**觸發**：外部架構 review 指出五個問題（品質閘門自評、無併發鎖、進化閾值過高、$0 算力浪費、死代碼）。

**核心轉變**：
```
舊思維：16GB 很窮，能省則省 → 減法設計 → 越做越小
新思維：16GB + Gemini Free + Claude Max + GPT Plus = 四台 $0 引擎 → 加法設計 → 越做越強
```

### 避坑指南

**一定要做**：
1. `think: False` 放 top-level（不是 `options` 裡面）
2. 用 `LLMResult.success` 檢查錯誤
3. 用 `dispatcher.call()` 按任務類型路由
4. 用 `caller` 參數追蹤 token 消耗
5. 繁中用 `tc=True` 啟用 opencc

**不要做**：
1. 不要把 Gemini 當品質閘門
2. 不要同時拉多個大模型（16GB 硬限制）
3. 不要手動 `ollama pull`（用 `ollama-update.sh`）
4. 不要跳過 `--use-llm`（instinct-engine 必傳）

### 回滾計畫

```bash
# models.py v2 回滾（chezmoi 追蹤）
cd $(chezmoi source-path) && git log --oneline -5
git checkout HEAD~1 -- dot_claude/skills/_lib/models.py
chezmoi apply

# 地端模型回滾
ollama pull qwen3.5:9b@sha256:6488c96fa5fa
```
