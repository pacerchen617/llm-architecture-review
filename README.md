# LLM 架構全景 — 2026-04-05 單模型統一版

> 本文件供跨 agent review 與學習。包含背景脈絡、決策歷程、當前配置、API 用法。
> 最後更新：2026-04-05 21:50 GMT+8

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

雲端 Opus/GPT-5.4 很強，但有三個問題：
1. **延遲**：每次 3-5 秒，自動化 pipeline 每天跑幾十次，累計太慢
2. **Token 消耗**：排程任務（分類、品質檢查、摘要）佔掉主 agent 的 context window
3. **依賴性**：網路斷了 = pipeline 停擺

所以需要一個**地端 LLM** 處理「不需要 Opus 級智商」的重複性任務。

### 原始架構（已淘汰）

```
雙模型架構（2026-03 ~ 2026-04-05）
├── Tier 0: qwen3:1.7b (1.4GB) — 分類/路由/品質評分，永駐
├── Tier 1: gemma4:e4b (9.6GB) — 繁中生成/摘要，按需載入
└── Embedding: qwen3-embedding:0.6b (639MB) — Vault Search
```

**痛點**：
- 16GB RAM 無法同時載入兩個模型（1.9GB + 10.7GB = 12.6GB > 可用 RAM）
- 需要序列化載入，增加管理複雜度
- `qwen3:1.7b` 繁中品質差，生成會混簡體
- `qwen3:1.7b` 的 `format_json=True` 會回空 JSON（已知 bug）

---

## 1. 決策歷程：怎麼走到現在的架構

### Round 1 Benchmark — 發現公平性問題

**日期**：2026-04-05
**候選**：Gemma4 E4B vs Qwen3.5 9B
**測試**：10+1 題合成任務

| | Gemma4 E4B | Qwen3.5 9B |
|---|---|---|
| 自動評分 | 33/35 | 12/35 |
| 平均延遲 | 48s | 87s |
| 空回覆 | 0/11 | **8/11** |

Qwen3.5 看似全面慘敗，但 root cause 分析發現是**測試參數不公平**：

- `num_predict=1024` 太小 — Qwen3.5 的 thinking tokens 和 output tokens **共用預算**，thinking 就用完了
- `think=false` 放在 `options` 裡 — Ollama `/api/generate` 有 bug（Issue #14793），**必須放在 top-level**
- 結論：**Round 1 對 Qwen3.5 不公平**，需要重新設計

### Round 2 Benchmark — 公平對決

**修正的參數**：
- `num_predict`: 1024 → **4096**
- `think`: 未設定 → **false (top-level)**
- `num_ctx`: 預設 → **8192**
- 任務來源：合成 → **25 題真實 pipeline prompt**

25 題來自六個真實 pipeline（instinct-engine 分類/摘要、learning-log-utils 品質評分、curiosity-loop 來源評估、evolution-loop 活動評分、筆記分類），不是虛構的測試題。

**結果**：

| 指標 | Gemma4 E4B | Qwen3.5 9B | 差異 |
|------|-----------|-----------|------|
| 品質 | **4.64/5** | 4.48/5 | Gemma 高 3.5% |
| 速度（25題） | 201s | **157s** | Qwen 快 22% |
| 磁碟 | 9.6 GB | **6.6 GB** | Qwen 省 31% |
| RAM 峰值 | ~12.6 GB | **~7.2 GB** | Qwen 省 43% |

**分項比較**：
- 分類任務：Gemma4 準確率較高（4.5 vs 4.0），但 Qwen3.5 快 2.5 倍（1.7s vs 4.3s/題）
- 繁中摘要：兩者都 5.0/5 滿分，Qwen3.5 快 2 倍
- 筆記分類：Qwen3.5 反超（4.6 vs 4.4）

### 最終決策

**選 Qwen3.5 9B 作為唯一地端模型**，理由：

1. **Pipeline 場景速度優先** — 每天跑幾十次的排程任務，22% 速度差比 3.5% 品質差重要
2. **RAM 釋放** — 單模型 7.2GB，不再有 OOM 和序列化問題
3. **管理簡化** — 一個模型 = 零切換邏輯，少一半故障點
4. **繁中品質足夠** — Round 2 摘要滿分，不再需要 gemma4 專門生成繁中

**淘汰條件**（什麼時候該換）：
- 品質 regression 超過 10% → 回退雙模型
- 出現更好的 <10GB 繁中模型 → 評估替換
- Ollama 修復 `/api/generate` 的 `think` 參數 bug → 移除 workaround

---

## 2. 當前架構（Ground Truth）

### 模型清單

| 模型 | 角色 | 大小 | 常駐策略 | 狀態 |
|------|------|------|---------|------|
| `qwen3.5:9b` | 統一：分類/路由/品質檢查/摘要/繁中生成 | 6.6 GB | `keep_alive=30m` | ✅ 生產 |
| `qwen3-embedding:0.6b` | Vault Search embedding（語意搜尋） | 639 MB | `keep_alive=-1`（永駐） | ✅ 生產 |

**已淘汰**：`qwen3:1.7b` (2026-04-05)、`gemma4:e4b` (2026-04-05)、`qwen3.5:4b`、`qwen3:8b`

### 完整分層圖

```
┌─────────────────────────────────────────────────────────────┐
│                    Kai（人類，決策者）                         │
│              S/A 等級停等確認 · 方向校準                       │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│              Claude Opus（大腦 · Orchestrator）               │
│                                                             │
│  收到任務 → 拆解子任務 → 標記 mode+affinity → 派工            │
│  收到結果 → Review（僅 S/A 等級）→ 綜合 → 再規劃或交付        │
│                                                             │
│  【不做的事】：每步路由判斷、重複性執行、批量處理              │
│  【成本】：$0（Claude Max 訂閱，本機 CLI 直連）              │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│              地端 LLM 層（Ollama 0.20.2 + MLX）              │
│                                                             │
│  ┌─────────────────────────────────────────────┐            │
│  │ qwen3.5:9b (6.6GB) — 統一模型               │            │
│  │                                             │            │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐    │            │
│  │  │ 分類/路由 │ │ 品質閘門  │ │ 摘要/生成 │    │            │
│  │  │ ~200ms   │ │ ~200ms   │ │ ~200ms   │    │            │
│  │  └──────────┘ └──────────┘ └──────────┘    │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  ┌─────────────────────────────────────────────┐            │
│  │ qwen3-embedding:0.6b (639MB) — 永駐         │            │
│  │ Vault Search 語意 embedding                  │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  API 端點：http://localhost:11434                            │
│  Runtime：Apple Silicon MPS 加速                             │
└──────────────────────────┬──────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                    雲端 LLM（按需）                           │
│                                                             │
│  Gemini 2.5 Flash  — 長文摘要/多模態/批量（$0 Free API）     │
│  Claude Opus       — 複雜推理/code gen/架構設計（$0 Max）     │
│  GPT-5.4           — 數據分析/教練邏輯（$0 ChatGPT Plus）    │
│  Claude Sonnet     — 輕量雲端任務                            │
└─────────────────────────────────────────────────────────────┘
```

### 路由表

| 任務類型 | 走哪裡 | 理由 |
|---------|--------|------|
| 文字分類/標籤/路由判斷 | qwen3.5:9b | 統一模型，~200ms |
| 品質評分（1-5 分） | qwen3.5:9b | rubric prompt |
| 繁中摘要/文案生成 | qwen3.5:9b | Round 2 驗證 5.0/5 |
| JSON 結構化輸出 | qwen3.5:9b | 格式精確 |
| Vault 語意搜尋 | qwen3-embedding:0.6b | 專用 embedding |
| 長文摘要（>10K 字） | Gemini 2.5 Flash | 1M context, $0 |
| 圖片描述/多模態 | Gemini 2.5 Flash | 多模態能力 |
| 複雜推理/架構設計/code gen | Claude Opus | 最強推理 |
| 數據分析/訓練計畫 | GPT-5.4 | 思考鏈 |

### 資源消耗

| 狀態 | RAM | 磁碟 |
|------|-----|------|
| Idle（只有 embedding） | ~1.4 GB | 6.7 GB |
| 推理中（兩個都載入） | ~7.2 GB | 6.7 GB |
| 系統剩餘（16GB Mac） | ~8.8 GB | — |

---

## 3. 誰在用這些 LLM

### Pipeline 清單

| Pipeline | 腳本 | 用了什麼 | 怎麼用 | 頻率 |
|----------|------|---------|--------|------|
| Instinct Engine | `instinct-engine.py` v3.1 | `call_ollama()` → qwen3.5:9b | 分類觀察類型 + 生成繁中摘要 | daily 03:00 + 每次 compact |
| Quality Gate | `_lib/models.py` | `quality_gate()` → qwen3.5:9b | 新 instinct/lesson 品質檢查（1-5 分） | 每次知識寫入前 |
| Cognize | `learning-log-utils.py` v9 | `quality_gate()` → qwen3.5:9b | 灰色地帶（noise 0.3-0.6）二次評分 | ~5 次/日 |
| Curiosity Loop | `curiosity-loop.py` | CLIProxyAPI → claude-sonnet-4-6 | 外部來源品質評估（**不走 Ollama**） | daily 04:30 |
| Evolution Loop | `evolution-loop.py` | CLIProxyAPI → claude-sonnet-4-6 | 活動評分 + 假說生成（**不走 Ollama**） | weekly |
| Vault Search | Obsidian plugin | `qwen3-embedding:0.6b` | 筆記語意 embedding | 用戶搜尋時 |

> ⚠️ **注意**：Curiosity Loop 和 Evolution Loop 走的是 CLIProxyAPI（localhost:8317）+ Claude Sonnet，不是地端 Ollama。這兩個 pipeline 的 LLM 成本是 $0（Claude Max 額度），但不受 Ollama 架構變更影響。

### LLM 極少化成果

平行進行的另一個專案：盤點 58 個排程任務的 LLM 依賴度，能用 heuristic/template/rule 就不燒 LLM。

| 批次 | 做了什麼 | 狀態 |
|------|---------|------|
| Batch 1 | workflow-audit 清理 LLM 死碼 + TG 追蹤迴圈 | ✅ done |
| Batch 2 | insight-router 8 組 keyword pattern bypass + curiosity-loop template 化 | ✅ done |
| Batch 3 | evolution-loop 7 維信號 gate + daily-intelligence auto-skip | ✅ done |
| Batch 4 | Threads LLM 腳本 + Meta Ads 評估 | ⏳ pending |

目前分類：**~35 個任務已無 LLM** / 5 個可降級 / 4 個必須 LLM

---

## 4. 品質閘門（Quality Gate）

所有知識寫入記憶系統前，必經雙層品質檢查：

```
輸入（新 lesson / 新 instinct）
  ↓
Layer 1: Heuristic Noise Filter（0ms，純正則+規則）
  ├─ score < 0.3 → 直接 REJECT（明顯 noise）
  ├─ score 0.3-0.6 → 灰色地帶 → 進入 Layer 2
  └─ score > 0.6 → 直接 PASS
  ↓
Layer 2: LLM Quality Gate（~200ms，qwen3.5:9b）
  ├─ score ≤ 2 → REJECT
  ├─ score = 3 → ADD + 標記 review flag
  └─ score ≥ 4 → PASS
```

**設計邏輯**：
- 不全走 LLM → 每次 ~200ms，daily-backup 批次跑太慢
- Heuristic 先擋明確的 → 只有灰色地帶才呼叫 LLM
- user/human source 永遠 bypass 兩層 → 人的輸入不需要 AI 判斷
- 單模型 self-judge → Round 2 Benchmark 驗證品質閘門分數合理

**實測數據**（2026-04-05）：
```
"Config was changed today"     → heuristic 0.590 (grey) → LLM score=1 → REJECT ✅
"The test worked as expected"  → heuristic 0.593 (grey) → LLM score=2 → REJECT ✅
"Restarted the main service"   → heuristic 0.592 (grey) → LLM score=1 → REJECT ✅
instinct-engine digest: 3 rejected, 1 borderline, 4 passed ✅
```

---

## 5. API 用法（給開發者）

### 統一入口

所有地端 LLM 呼叫走同一個 Python module：

```python
# 檔案位置：~/.claude/skills/_lib/models.py

from _lib.models import call_ollama, call_ollama_chat, quality_gate, ensure_ollama_model
```

### call_ollama — 單輪對話

```python
result = call_ollama(
    prompt="分類這段文字：...",
    model="qwen3.5:9b",       # 預設值，通常不需指定
    temperature=0.3,           # 預設值
    num_ctx=4096,              # 預設值
    timeout_sec=120,           # 預設值
    caller="my-script",        # 用途追蹤（寫入 usage log）
    tc=False                   # True = 自動 opencc 簡轉繁（s2twp）
)
# 回傳：str（模型回覆）或 "ERROR: ..." 字串
```

### call_ollama_chat — 多輪/角色對話

```python
result = call_ollama_chat(
    messages=[
        {"role": "system", "content": "你是分類器，只回一個詞"},
        {"role": "user", "content": "分類這段..."}
    ],
    model="qwen3.5:9b",
    caller="my-script"
)
```

### quality_gate — 品質評分

```python
qg = quality_gate(
    output="模型生成的文字",
    task="這段文字的任務描述",
    judge_model="qwen3.5:9b"   # 預設值
)
# 回傳：{"score": 4, "reason": "一句話理由", "pass": True}
# score ≥ 4 = pass (pass=True)
# score = 3 = pass=False，但 caller 可自行決定「加入 + 標 review flag」
# score ≤ 2 = pass=False，建議 reject 或升級到 Opus
# 注意：score=3 的「review flag」邏輯在 caller 端實作（如 learning-log-utils），不在 quality_gate 內
```

### ensure_ollama_model — 預熱/保持模型載入

```python
ensure_ollama_model(
    model="qwen3.5:9b",
    keep_alive="-1"    # "-1" = 永駐, "30m" = 30 分鐘後卸載
)
# 用途：確保模型已載入 RAM，避免第一次呼叫的冷啟動延遲
# 如果模型未安裝，會回傳 error 但不會自動 pull
```

### 關鍵實作細節

| 細節 | 說明 | 為什麼重要 |
|------|------|-----------|
| `think: False` | payload **top-level**，不是 `options` 裡面 | qwen3.5 預設開 thinking mode，不關 → 所有 token 花在思考，response 回空字串 |
| `tc=True` | 自動走 opencc s2twp（簡體→繁體+台灣用語） | qwen3.5 繁中品質已驗證（5.0/5），opencc 當保險 |
| `format_json` | 有支援，但注意 output 格式 | 舊 qwen3:1.7b 有空 JSON bug，qwen3.5 無此問題 |
| error handling | 所有函數失敗回傳 `"ERROR: ..."` 字串 | caller 需要檢查 `result.startswith("ERROR")` |
| usage logging | 自動記錄 `prompt_eval_count` + `eval_count` | 追蹤 token 消耗，在 usage log 裡 |

### OLLAMA_MODELS dict

```python
OLLAMA_MODELS = {
    "router": "qwen3.5:9b",           # 統一模型
    "writer": "qwen3.5:9b",           # 同 router（單模型架構）
    "embed":  "qwen3-embedding:0.6b",  # embedding，不變
}
```

可以用 key 或完整模型名呼叫：`call_ollama(prompt, model="router")` 等同 `model="qwen3.5:9b"`。

---

## 6. 跨 Agent 影響

| Agent | 影響 | 細節 |
|-------|------|------|
| 🏭 夢工廠 | ✅ 直接受益 | instinct-engine、quality-gate、learning-log-utils 都走新模型 |
| 📊 投放者 | ⚠️ 間接 | 如果 import `_lib/models.py`，自動走 qwen3.5:9b |
| 🧵 社群飛輪 | ❌ 無影響 | 走 GPT-5.4（ChatGPT Plus OAuth），不用地端 LLM |
| 🧗 體能工作室 | ❌ 無影響 | 走 GPT-5.4（ChatGPT Plus OAuth），不用地端 LLM |

**共用資源**：
- `~/.claude/skills/_lib/models.py` — 所有 agent 共用的 Ollama API wrapper
- Ollama 服務（port 11434）— 同一台機器，同一個 instance
- 16GB RAM — 所有 agent + LLM + OS 共用

---

## 7. 避坑指南

### 一定要做

1. **`think: False` 放 top-level** — 不是 `options` 裡面，不是 `messages` 裡面
2. **檢查 error 回傳** — `result.startswith("ERROR")` 是唯一錯誤判斷方式
3. **用 `caller` 參數** — 方便追蹤哪個腳本在燒 token
4. **繁中用 `tc=True`** — opencc 自動轉換，不需要手動處理

### 不要做

1. **不要 `think: True`** — 除非你真的需要 reasoning chain，否則白花 token
2. **不要改 embedding 模型** — `qwen3-embedding:0.6b` 和 qwen3.5 是不同用途
3. **不要同時拉大模型** — 16GB RAM 限制，一個 9B 已經是極限
4. **不要在 `options` 裡放 `think`** — Ollama `/api/generate` 的已知 bug（Issue #14793）

### 歷史教訓

| 事件 | 教訓 |
|------|------|
| Round 1 Qwen3.5 8/11 空回覆 | `num_predict` 太小 + `think` 參數位置錯 = 看起來模型很爛但其實是配置問題 |
| qwen3:1.7b `format_json` 回空 JSON | 小模型的 JSON 能力不可靠，用自然語言 + regex 解析更穩 |
| 雙模型 RAM 互斥 | 16GB 機器上不要貪心跑兩個大模型，單模型簡單穩定 |
| 品質 4.48 vs 4.64 差 3.5% | Pipeline 場景下速度和穩定性比微小品質差異更重要 |

---

## 8. 回滾計畫

如果 qwen3.5:9b 出重大問題：

```bash
# Step 1: 拉回舊模型
ollama pull gemma4:e4b
ollama pull qwen3:1.7b

# Step 2: 回退程式碼
cd $(chezmoi source-path) && git log --oneline -5
# 找到 "LLM migration" commit，revert 它
chezmoi apply

# Step 3: 驗證
grep -n "qwen3.5:9b" ~/.claude/skills/_lib/models.py  # 應該消失
ollama list  # 應該有三個模型
```

---

## 9. 相關檔案索引

| 用途 | 路徑 |
|------|------|
| API 實作 | `~/.claude/skills/_lib/models.py` |
| Instinct Engine | `~/.local/bin/instinct-engine.py` |
| Learning Log Utils | `~/.local/bin/learning-log-utils.py` |
| Round 1 結果 | `~/.claude-bot-memory/llm-benchmark-round1-results.md` |
| Round 2 結果 | `~/.claude-bot-memory/llm-benchmark-round2-results.md` |
| 遷移任務書 | `~/.claude-bot-memory/task-local-llm-consolidation.md` |
| 選型計畫 | `~/.claude-bot-memory/plan-2026-04-05-local-llm-selection.md` |
| Obsidian 架構圖 | `Vault/Architecture/Multi-Model Ecosystem.md` |
| Obsidian 品質閘門 | `Vault/Architecture/Quality Gate System.md` |
| Obsidian 服務 | `Vault/Services/Ollama Local AI.md` |
| strategies.json | `~/.claude-bot-memory/strategies.json` → `local_model_routing` |
| 極少化任務 | `active-tasks.json` → `llm-minimization-2026` |
