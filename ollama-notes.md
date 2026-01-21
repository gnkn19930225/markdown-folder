# Ollama 學習筆記

## 常用指令

| 指令 | 說明 |
|------|------|
| `ollama list` | 列出本機所有模型 |
| `ollama ps` | 查看目前執行中的模型（含記憶體佔用） |
| `ollama run <model>` | 執行模型進入對話 |
| `ollama pull <model>` | 下載模型 |
| `ollama rm <model>` | 刪除模型 |
| `ollama show <model>` | 顯示模型詳細資訊（含 Modelfile） |
| `ollama cp <src> <dst>` | 複製模型 |
| `ollama create <name> -f <Modelfile>` | 從 Modelfile 建立模型 |

### 對話中指令

| 指令 | 說明 |
|------|------|
| `/save <name>` | 儲存目前狀態為新模型 |
| `/show info` | 顯示模型資訊 |
| `/set parameter <name> <value>` | 臨時修改參數 |
| `/clear` | 清除對話歷史 |
| `/bye` | 離開對話 |

---

## Modelfile（模型設定檔）

用於定義自訂模型，可包含 system prompt、參數設定等。

### 基本結構

```dockerfile
# 指定基底模型
FROM llama3

# 設定 system prompt
SYSTEM """
你是一個專業的程式助手，請用繁體中文回答。
"""

# 調整參數
PARAMETER temperature 0.7
PARAMETER top_p 0.9
PARAMETER num_ctx 4096
```

### 常用 PARAMETER 參數

| 參數 | 說明 | 預設值 |
|------|------|--------|
| `temperature` | 創意度（越高越隨機） | 0.8 |
| `top_p` | 取樣範圍 | 0.9 |
| `top_k` | 取樣候選數 | 40 |
| `num_ctx` | 上下文長度 | 2048 |
| `repeat_penalty` | 重複懲罰 | 1.1 |

### 建立自訂模型

```bash
# 從 Modelfile 建立模型
ollama create my-assistant -f ./Modelfile
# transferring model data
# creating model layer
# writing manifest
# success

# 驗證
ollama list
# NAME              ID            SIZE      MODIFIED
# my-assistant      a]b2c3d4e5    4.7 GB    just now
# llama3            f6g7h8i9j0    4.7 GB    2 days ago

# 執行
ollama run my-assistant
# >>> 輸入訊息開始對話...
```

---

## 儲存對話為新模型

另一種建立自訂模型的方式：在對話中直接儲存。

```bash
# 進入對話
ollama run llama3
# >>>

# 設定 system prompt 或對話後，使用 /save 儲存
>>> /save my-custom-model
# Created new model 'my-custom-model'

# 驗證
ollama list
# NAME              ID            SIZE      MODIFIED
# my-custom-model   x1y2z3a4b5    4.7 GB    just now
# llama3            f6g7h8i9j0    4.7 GB    2 days ago

# 之後可直接使用
ollama run my-custom-model
# >>> 輸入訊息開始對話...
```

---

## RAG 應用

Modelfile 可搭配 RAG（檢索增強生成）使用，透過 SYSTEM prompt 定義檢索行為。

### RAG 用 Modelfile 範例

```dockerfile
FROM llama3

SYSTEM """
你是一個知識庫助手。請根據提供的上下文資料回答問題。
規則：
1. 只使用提供的資料回答
2. 如果資料中沒有相關內容，請誠實說明「資料中未提及」
3. 回答時引用資料來源
"""

PARAMETER temperature 0.3
PARAMETER top_p 0.9
```

---

## 模型大小計算

### 參數量（B = Billion）

模型名稱中的 `7b`、`8b`、`13b`、`70b` 代表參數量（十億）。

### 計算公式

```
模型大小 = 參數量(B) × 每參數位元組數
```

### 精度對應位元組數

| 精度 | 每參數位元組數 | 說明 |
|------|----------------|------|
| FP32 | 4 bytes | 全精度浮點數 |
| FP16 / BF16 | 2 bytes | 半精度浮點數 |
| INT8 (Q8) | 1 byte | 8-bit 量化 |
| INT4 (Q4) | 0.5 byte | 4-bit 量化 |

### 模型大小換算表

| 參數量 | FP32 (×4) | FP16 (×2) | INT8 (×1) | INT4 (×0.5) |
|--------|-----------|-----------|-----------|-------------|
| **7B** | 7 × 4 = **28 GB** | 7 × 2 = **14 GB** | 7 × 1 = **7 GB** | 7 × 0.5 = **3.5 GB** |
| **8B** | 8 × 4 = **32 GB** | 8 × 2 = **16 GB** | 8 × 1 = **8 GB** | 8 × 0.5 = **4 GB** |
| **13B** | 13 × 4 = **52 GB** | 13 × 2 = **26 GB** | 13 × 1 = **13 GB** | 13 × 0.5 = **6.5 GB** |
| **32B** | 32 × 4 = **128 GB** | 32 × 2 = **64 GB** | 32 × 1 = **32 GB** | 32 × 0.5 = **16 GB** |
| **70B** | 70 × 4 = **280 GB** | 70 × 2 = **140 GB** | 70 × 1 = **70 GB** | 70 × 0.5 = **35 GB** |

> **注意**：實際大小會因模型架構、量化方法略有差異，上表為理論估算值。
