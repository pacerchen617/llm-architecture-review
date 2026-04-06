# LLM 架構全景 — 2026-04-06 四層算力池版

> 本文件供跨 agent review 與學習。包含背景脈絡、決策歷程、當前配置、API 用法。
> 最後更新：2026-04-06 10:00 GMT+8

---

## 0. 背景：我們在解決什麼問題

### 系統概覽

五個 AI agent 共用一台 16GB MacBook，各自有不同的 LLM 需求：

| Agent | 主要模型 | 用途 |
|-------|---------|------|
| 🏭 夢工廠 | Claude Opus ($0 Claude Max) | 程式開發/系統管理 |
| 📊 投放者 | Claude Opus ($0 Claude Max) | Meta 廣告管理 |
| 🧵 社群飛輪 | GPT-5.4 ($0 ChatGPT Plus) | Threads 社群經營 |
| 🧗 體能工作室 | GPT-5.4 ($0 ChatGPT Plus) | 耐力運動教練 |
| 🦞 蝦蝦 | 已退休 (2026-04-05) | — |

**核心原則**：$0 增量成本。所有 LLM 都走已付費的訂閱額度或免費 API。

### 為什麼需要地端 LLM

雲端 Opus/GPT-5.4 很強（都是 1M context），但有三個問題：
1. **延遲**：每次 3-5 秒，自動化 pipeline 每天跑幾十次，累計太慢
2. **Token 消耗**：排程任務（分類、品質檢查、摘要）佔掉主 agent 的 context window
3. **依賴性**：網路斷了 = pipeline 停擺

所以需要一個**地端 LLM** 處理「不需要 Opus 級智商」的重複性任務。雲端強模型留給品質判斷和深度推理。

---

## 1. 決策歷程

### Round 1 → Round 2 Benchmark（詳見舊版 README）

簡要：Qwen3.5 9B 在公平 Round 2 測試中勝出 — 速度快 22%、省 31% RAM、繁中品質足夠（4.48/5）。

### 2026-04-06 架構升級：從「單模型省著用」到「四層算力池」

**觸發原因**：外部架構 review（Kai vs Tobia 對比文件）指出：
- 品質閘門用同一模型自己評自己 = **學生改自己考卷**
- Ollama 無併發鎖 = 多 agent 同時呼叫可能 OOM
- 進化閾值需要 32 次觀察才畢業 = **管路從未通過**
- Gemini/Claude 只當 fallback = **浪費已有的 $0 算力**
- daily-backup 呼叫 instinct-engine 時未傳 `--use-llm` = **LLM 路徑是死代碼**

**核心轉變**：
```
舊思維：16GB 很窮，能省則省 → 減法設計 → 越做越小
新思維：16GB + Gemini Free + Claude Max + GPT Plus = 四台 $0 引擎 → 加法設計 → 越做越強
```

---

## 2. 當前架構（2026-04-06 Ground Truth）

### 四層算力池

```
Layer 0 即時層  │ qwen3.5:9b 本地 ($0, ~200ms, 離線可用, 無限量)
               │ → 分類/路由/初篩/快速生成
               │
Layer 1 品質層  │ Sonnet 4.6 via CLIProxyAPI ($0 Claude Max, 1M ctx, 64K out)
               │ → 品質閘門 second opinion / 高頻摘要 / instinct 分類
               │
Layer 2 深度層  │ Opus 4.6 via CLIProxyAPI ($0 Claude Max, 1M ctx, 128K out)
               │ → L2 知識聚合 / 記憶衝突裁決 / 複雜推理
               │
Layer 3 獨家能力 │ Gemini Flash ($0 Free, Google Search grounding)
               │ → 即時搜尋研究 / grounding 標注
               │ GPT-5.4 ($0 ChatGPT Plus, 1M ctx)
               │ → 跨架構交叉驗證

         ┌───────────────────────────────────────────────┐
未來:     │ Mac Mini M4 Pro 64GB → Layer 0 升級 26B+ 模型  │
         │ M3 36GB MBP → Layer 0 備援 + 模型實驗場       │
         └───────────────────────────────────────────────┘
```

### 模型清單

| 模型 | Context | Output | 費用 | 角色 | 獨家能力 |
|------|---------|--------|------|------|---------|
| Claude Opus 4.6 | 1M | 128K | $0 Max | L2 深度推理 | 最強推理+超長輸出 |
| Claude Sonnet 4.6 | 1M | 64K | $0 Max | L1 品質閘門 | 高速強推理 |
| GPT-5.4 | 1M | ? | $0 Plus | L3 跨架構驗證 | 不同架構=不同盲點 |
| Gemini Flash | 1M | 65K | $0 Free | L3 Research | Google Search grounding |
| qwen3.5:9b | 32K | — | 本地 | L0 即時層 | 離線+零延遲+無限量 |
| qwen3-embed:0.6b | — | — | 本地 | Embedding | Vault Search 專用 |

**已淘汰**：qwen3:1.7b, gemma4:e4b, qwen3.5:4b, qwen3:8b

### 路由表（ModelDispatcher）

| 任務類型 | 主力 | 備援 | 為什麼 |
|---------|------|------|--------|
| 品質判斷 (judge) | Sonnet 4.6 | Opus 4.6 | 不同模型=真 second opinion |
| 深度推理 (reason) | Opus 4.6 | Sonnet 4.6 | 最強推理做最難的事 |
| 即時分類 (classify) | qwen3.5:9b | Sonnet 4.6 | 本地 0ms，掛了才走雲端 |
| 摘要 (summarize) | Sonnet 4.6 | qwen3.5:9b | 品質重要 |
| 研究 (research) | Gemini + grounding | Sonnet 4.6 | 唯一有 Google Search |
| 知識聚合 (aggregate) | Opus 4.6 | Sonnet 4.6 | 1M context + 128K output |
| 跨架構驗證 (validate) | GPT-5.4 | Opus 4.6 | 不同架構 |
| Embedding (embed) | qwen3-embed:0.6b | — | 唯一選擇 |
| 一般生成 (generate) | qwen3.5:9b | Sonnet 4.6 | 本地優先 |

### 資源消耗

| 狀態 | RAM | 磁碟 |
|------|-----|------|
| Idle（只有 embedding） | ~1.4 GB | 6.7 GB |
| 推理中（qwen3.5 + embedding） | ~7.2 GB | 6.7 GB |
| 系統剩餘（16GB Mac） | ~8.8 GB | — |

---

## 3. 品質閘門 v2（三層）

```
輸入（新 lesson / 新 instinct）
  ↓
Layer 1: Heuristic Noise Filter（0ms，純正則+規則）
  ├─ score < 0.3 → 直接 REJECT
  ├─ score 0.3-0.6 → 灰色地帶 → Layer 2
  └─ score > 0.6 → 直接 PASS
  ↓
Layer 2: qwen3.5:9b 本地初篩（~200ms）
  ├─ score ≤ 2 → REJECT
  ├─ score = 3 → 灰色地帶 → Layer 3（Sonnet second opinion）
  └─ score ≥ 4 → PASS
  ↓
Layer 3: Sonnet 4.6 Second Opinion（灰色地帶才觸發）
  ├─ 取兩者較低分（保守策略）
  ├─ 兩者一致 → 直接採用
  └─ Sonnet 不可用 → 回退用 L2 分數
```

**為什麼品質閘門用 Sonnet 而不是 Opus？**
- 品質閘門是高頻操作（每日 30-50 次）
- Sonnet 品質夠強，速度更快
- Opus 留給最高價值任務（L2 聚合、衝突裁決、複雜推理）
- Max 方案有尖峰時段限額風險，需分散壓力

**為什麼不用 Gemini Flash？**
- Opus > Sonnet >> Gemini Flash 在判斷品質上
- 我們有 $0 Claude Max，品質判斷用最強的，不用省
- Gemini 的正確用途是它的獨家能力：Google Search grounding

---

## 4. Instinct Pipeline（已修復）

```
PostToolUse hook → auto-observe.sh → reflect-queue.jsonl
                                         ↓
daily-backup 03:00 → instinct-engine digest --use-llm
                                         ↓
                     qwen3.5:9b 分類 + 摘要 + quality_gate
                                         ↓ (score ≥ 3)
                     instincts.jsonl → evolve (weekly)
                                         ↓ (confidence ≥ 0.65)
                     learning-log.jsonl → Obsidian Vault
```

**2026-04-06 修復清單**：
- `--use-llm` 加入 daily-backup（原本 LLM 路徑是死代碼）
- 進化閾值：0.8 → 0.65（原本需要 32 次觀察，現在 8 次即可接近）
- 信心公式：`0.1 * log₂(n)` → `0.15 * log₂(n)`
- quality_gate 灰色地帶送 Sonnet second opinion

---

## 5. 知識聚合 L2（新增）

每週日 `memory-weekly-compound.sh` Phase 3.5 觸發：

```
本週 learning-log + instincts + active-tasks
  ↓
weekly-trend-l2.py → Opus 4.6（1M context 全局俯瞰）
  ↓
Obsidian Vault/Learnings/weekly-trend-YYYY-WW.md
  ├─ 跨類別趨勢
  ├─ 重複模式偵測
  ├─ 知識缺口識別
  └─ 成長建議
```

---

## 6. API 用法

### 新版：LLMResult + ModelDispatcher

```python
from _lib.models import (
    LLMResult,          # 統一回傳類型
    ModelDispatcher,    # 任務導向路由
    dispatcher,         # singleton instance
    call_model,         # Claude via CLIProxyAPI → returns LLMResult
    call_ollama,        # Local Ollama → returns string（向後兼容）
    call_ollama_result, # Local Ollama → returns LLMResult
    call_gemini,        # Gemini API → returns LLMResult
    call_codex,         # GPT via Codex CLI → returns LLMResult
    quality_gate,       # 三層品質閘門 → returns dict
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

# 用法
result = call_model("claude-sonnet-4-6", "system prompt", "user input")
if result.success:
    print(result.content)
else:
    print(f"Failed: {result.error}")

# 向後兼容：str(result) 回傳 content 或 "ERROR: ..."
text = str(result)
```

### ModelDispatcher — 按任務性質自動路由

```python
# 品質判斷 → Sonnet → Opus
result = dispatcher.call("評估這段輸出", task_type="judge")

# 研究 → Gemini + grounding → Sonnet
result = dispatcher.call("搜尋最新...", task_type="research")

# 深度推理 → Opus → Sonnet  
result = dispatcher.call("分析這個架構", task_type="reason")

# 自動 fallback：主力失敗自動切備援
```

### call_ollama（向後兼容）

```python
result = call_ollama(
    prompt="分類這段文字：...",
    model="qwen3.5:9b",
    caller="my-script",
    tc=True                 # opencc 簡轉繁
)
# 回傳 string，與舊版 API 完全相同
# 內部已加 fcntl.flock 防止併發 OOM
```

### quality_gate v2

```python
qg = quality_gate(output="...", task="...")
# 回傳：{"score": 4, "reason": "...", "pass": True, "judge": "qwen3.5:9b+sonnet-4.6"}
#   score ≥ 4: pass（L1 直接通過或 L2 確認通過）
#   score = 3: 經 Sonnet second opinion 後決定
#   score ≤ 2: reject
```

### call_gemini（新增 grounding 支援）

```python
result = call_gemini(
    prompt="搜尋最新的...",
    grounding=True,          # 啟用 Google Search grounding
    model="gemini-2.5-flash"
)
```

---

## 7. 併發安全 + 版本釘選

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

`ollama-update.sh`（每週日 04:00）會檢查 digest，pinned 模型**不會被自動更新**。版本變更時通知 TG。

---

## 8. 跨 Agent 影響

| Agent | 影響 | 細節 |
|-------|------|------|
| 🏭 夢工廠 | ✅ 直接受益 | 四層算力池、ModelDispatcher、quality_gate v2 |
| 📊 投放者 | ⚠️ 間接 | import `_lib/models.py` 自動獲得新能力 |
| 🧵 社群飛輪 | ❌ 無影響 | 走 GPT-5.4，不用地端 LLM |
| 🧗 體能工作室 | ❌ 無影響 | 走 GPT-5.4，不用地端 LLM |

**Agent Mailbox**（新增）：4 個活躍 agent 已初始化 mailbox（`~/.openclaw/mailboxes/`），支援結構化跨 agent 通信。

---

## 9. 避坑指南

### 一定要做

1. **`think: False` 放 top-level** — 不是 `options` 裡面
2. **用 LLMResult.success 檢查錯誤** — 不再用 `startswith("ERROR")`
3. **用 `dispatcher.call()` 按任務類型路由** — 不要硬寫 backend
4. **用 `caller` 參數** — 追蹤 token 消耗來源
5. **繁中用 `tc=True`** — opencc 自動轉換

### 不要做

1. **不要把 Gemini 當品質閘門** — 品質判斷用 Sonnet/Opus，Gemini 只做 grounding
2. **不要同時拉大模型** — 有 flock 防護，但 16GB 仍然是硬限制
3. **不要手動 `ollama pull`** — 用 `ollama-update.sh` 走版本釘選流程
4. **不要跳過 `--use-llm`** — instinct-engine 不傳這個 flag = LLM 路徑全是死代碼

---

## 10. 回滾計畫

### models.py v2 回滾

```bash
# chezmoi 追蹤的，可以回退
cd $(chezmoi source-path) && git log --oneline -5
git checkout HEAD~1 -- dot_claude/skills/_lib/models.py
chezmoi apply
```

### 地端模型回滾

```bash
# 版本釘選，不會自動更新
# 如需手動回退：
ollama pull qwen3.5:9b@sha256:6488c96fa5fa
```

---

## 11. 相關檔案索引

| 用途 | 路徑 |
|------|------|
| API 實作（v2） | `~/.claude/skills/_lib/models.py` |
| Instinct Engine | `~/.local/bin/instinct-engine.py` |
| L2 Trend Analysis | `~/.local/bin/weekly-trend-l2.py` |
| Weekly Compound | `~/.local/bin/memory-weekly-compound.sh` |
| Ollama Update | `~/.local/bin/ollama-update.sh` |
| Model Versions | `~/.claude-bot-memory/model-versions.json` |
| Agent Mailbox | `~/.local/bin/agent-mailbox.py` |
| Round 2 結果 | `~/.claude-bot-memory/llm-benchmark-round2-results.md` |
| Obsidian 架構圖 | `Vault/Architecture/Multi-Model Ecosystem.md` |
| Obsidian 品質閘門 | `Vault/Architecture/Quality Gate System.md` |

---

## 12. 技術選型淘汰條件

| 選擇 | 為什麼 | 淘汰條件 |
|------|--------|---------|
| Sonnet 4.6 品質閘門 | 高頻+品質夠強 | Claude Max 取消訂閱 |
| Opus 4.6 L2 聚合 | 最強推理+128K output | 出現更強且$0的替代 |
| qwen3.5:9b 本地 | Round 2 驗證+省 RAM | Mac Mini 到位→升級 26B |
| Gemini Flash grounding | Google Search 獨家 | Claude 原生搜尋開放 |
| fcntl.flock | 單機簡單有效 | 多機部署→改分散式鎖 |
| JSONL 記憶 | 簡單+chezmoi 友好 | >5000 條→考慮 SQLite |
