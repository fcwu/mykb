---
title: 本地 LLM 推論：選型與硬體指南
type: concept
tags: [ai, llm, local-inference, moe, quantization, gemma, qwen, ollama, mlx, kv-cache, strix-halo, dgx-spark, amd, nvidia, apple-silicon, m5-pro]
created: 2026-04-07
updated: 2026-06-06
date: 2026-06-06
category: ai-agent
description: 2026 年消費級硬體跑大型語言模型的完整選型框架：KV Cache 原理、MoE 速度分析、Gemma vs Qwen 決策樹、M5 Pro / Strix Halo / DGX Spark 硬體 Prefill 實測比較。
---

# 本地 LLM 推論：選型與硬體指南

> 2026 年消費級硬體跑大型語言模型的完整選型框架，以 Gemma 4 vs Qwen 3.5 對決為核心案例。

---

## KV Cache 深度解析

### KV Cache 原理

- **作用**：存過去 token 的 K/V 向量，生成下一個 token 時不必重算整段序列，是「用記憶體換速度」
- **大小公式**：`2 × 層數 × context_length × KV_heads × head_dim × bytes_per_element`
- **預配時機**：KV cache 在**載入模型當下**就依 `context_length` 一次預配（preallocate），對話長了不會讓 VRAM 慢慢爬升；設越長的 context → 載入瞬間就吃越多
- **MoE 不省 KV cache**：attention 層是 dense、全部要算；KV cache 只跟 attention 結構有關，跟 active 參數量無關

### head_dim：KV cache 的隱藏放大器

| 模型 | head_dim | 相對 KV 大小 |
|------|----------|-------------|
| Qwen 系列 | 128 | 基準 |
| Gemma 4 sliding 層 | 256 | 2× |
| Gemma 4 global 層 | 512 | 4×（隨 context 線性成長） |

**結論**：Gemma 4 比 Qwen MoE 更吃 VRAM，不是因為 dense vs MoE，而是 head_dim 差距。Flash Attention 2（FA2）head_dim 上限是 256，Gemma 4 的 global 層需特別處理。

### LM Studio Unified KV Cache（軟體功能，非硬體）

並行請求共用同一個 KV cache 池，不是每個請求各切固定一塊。**與 AMD 統一記憶體硬體架構無關**（同名巧合）。

| 狀態 | 記憶體用量 | 特性 |
|------|-----------|------|
| 開啟 | `n_ctx × 1` | 省記憶體，並行時可能搶，單人建議開啟 |
| 關閉 | `n_ctx × n_seq_max`（預設 4） | 穩定，但浪費 |

### GTT（Graphics Translation Table）— AMD APU 專屬

讓 iGPU 把系統 RAM 當 VRAM 用的機制。適用 Strix Halo 等統一記憶體 APU。
- Windows：調 BIOS Variable Graphics Memory（VGM）
- Linux：調 `amdttm.pages_limit`（舊名：`amdgpu.gttsize`）

---

## 速度的根本原理：記憶體頻寬問題

**Decode 階段（batch=1）是 memory-bound，不是 compute-bound。**

```
理論 tok/s = VRAM 頻寬 ÷ 每 token 讀取量
每 token 讀取量 = 活躍參數 × bytes/param
```

實際值 ≈ 理論值 × 60–70%（Apple Silicon 稍高），差距來源：
- Q4 Dequantization overhead：~15–20% 額外計算
- MoE routing scatter/gather
- CUDA kernel launch latency（小 batch 最明顯）
- KV cache 讀取（context 越長影響越大）

**Prefill vs Decode**：prefill（處理 prompt）是 compute-bound，速度快但使用者感受不到。RAG 場景塞 10K token 時 TTFT 會長，但 decode 速度不受影響。

---

## 核心概念

### MoE（Mixture of Experts）的記憶體 vs 速度

- **記憶體**：MoE 的全部專家參數必須載入 VRAM——「活躍參數」只影響計算，不影響記憶體
  - Qwen 3.5 35B-A3B（3B 活躍）：Q4 需 ~24 GB VRAM
  - Gemma 4 26B-A4B（3.8B 活躍）：Q4 需 ~15–18 GB VRAM
- **速度優勢**：每次 forward pass 只用少數專家，速度遠快於同尺寸 Dense
  - RTX 4090：Qwen 3.5 35B-A3B MoE = 100+ tok/s；Qwen 3.5 27B Dense = ~35 tok/s

### 長上下文的 KV Cache 瓶頸

架構設計直接影響長上下文的記憶體佔用：

| 架構 | 代表模型 | RTX 4090 Q4 可用 Context |
|------|---------|------------------------|
| Gated DeltaNet（線性注意力） | Qwen 3.5 | ~190K tokens |
| Sliding-Window + Global Attention | Gemma 4 | ~20K tokens |
| 標準 GQA | 一般 27B+ 模型 | 8K–32K tokens |

**原因**：Gated DeltaNet 以固定記憶體維持線性複雜度，不像標準注意力隨 context 長度線性增長 KV cache。

#### Qwen 3.5 35B-A3B 的 KV Cache 計算

35B-A3B 的 KV cache 比直覺想的小，因為 **DeltaNet 層（75% 層）不產生 KV cache**，只有 GQA 層（25%）才有。

```
KV cache ≈ seq_len × 131 KB（FP16）
```

**RTX 4090 Q4_K_M（~24GB）的記憶體分配：**

| context | 模型 | KV cache | 系統 | 合計 | 備註 |
|---------|------|---------|------|------|------|
| 8K FP16 | 24 GB | 1.0 GB | 1.5 GB | ~26.5 GB | 稍緊但可跑 |
| 8K Q8 | 24 GB | 0.5 GB | 1.5 GB | ~26 GB | 建議預設 |
| 32K Q8 | 24 GB | 2.0 GB | 1.5 GB | ~27.5 GB | 需要 APEX Mini |
| 45K | 18 GB (Q3) | 5.7 GB | 1.5 GB | ~25 GB | Q3 + FP16 KV |

**延長 context 的策略優先順序：**
1. 開啟 **KV cache Q8 量化**（`-ctk q8_0 -ctv q8_0`）→ KV 省一半
2. 換用 **Q3_K_S / APEX Mini**（~18 GB）→ 空出 6 GB → ~45K context
3. 兩者並用 → KV Q4 → 32K+ context

```bash
# llama.cpp：Q8 KV，context 32K，全 GPU
llama-cli -m Qwen3.5-35B-A3B-Q4_K_M.gguf \
  -c 32768 -ctk q8_0 -ctv q8_0 --gpu-layers 99
```

> **注意**：Ollama 不支援手動設定 `-ctk`。需要精確控制 context 建議用 **llama.cpp** 或 **LM Studio**（GUI）。

### 量化等級選擇原則

| 場景 | 推薦量化 | 說明 |
|------|---------|------|
| 品質優先，VRAM 充裕 | Q8_0 / BF16 | 接近原始精度 |
| 平衡點（日常使用） | Q4_K_M | 體積最小化，品質損失小 |
| MoE 路由專家 | Q6_K | Q8 多 7.5GB 但 PPL 無改善 |
| 極限 VRAM 壓縮 | Q3_K_S / APEX Mini | 品質損失較大，但仍可用 |
| 最高品質 Q4 | Unsloth UD-Q4_K_XL | Dynamic 2.0，盲測優於標準 Q4_K_M |

---

## 主流模型對照（2026-04）

### 各 VRAM 區間最佳選擇

| GPU VRAM | 推薦模型 | 速度 | 品質 |
|----------|---------|------|------|
| 8–10 GB | Qwen 3.5 4B Q4 (3.4GB) | 高速 | 良好 |
| 12 GB | Qwen 3.5 9B Q4 (6.6GB) ⭐ | 高速 | 優秀 |
| 16 GB | Qwen 3.5 35B-A3B APEX Mini (12.2GB) | 高速 MoE | 優秀 |
| 24 GB | Qwen 3.5 35B-A3B Q4 ⭐⭐ | ~100 tok/s | 頂級 |
| 32 GB | Qwen 3.5 35B-A3B Q5 / Gemma 4 31B Q8 | 高速 | 頂級 |

**Qwen 3.5 9B 是隱藏冠軍**：MMLU-Pro 82.5%、GPQA Diamond 81.7%，接近 Gemma 4 26B-A4B MoE，僅需 6.6GB VRAM。

### Apple Silicon 選型

| RAM | 推薦配置 | Backend | 速度 |
|-----|---------|---------|------|
| 16 GB | Qwen 3.5 9B Q4 | MLX | ~40 tok/s |
| 36 GB | Qwen 3.5 27B Q4 | MLX | ~20 tok/s |
| 48–64 GB | Qwen 3.5 35B-A3B Q4 ⭐ | MLX | 60–70 tok/s |
| 128 GB | Qwen 3.5 122B-A10B Q4 (~81GB) | MLX | ~15 tok/s |

**MLX vs Ollama**：Apple Silicon 上 MLX 約快 2 倍。2026/03 起 Ollama 在 Mac 上預設使用 MLX backend。

---

## 本地推論框架

| 框架 | 特色 | 適用場景 |
|------|------|---------|
| **Ollama** | 一鍵安裝，GUI 友好 | 一般使用者，快速實驗 |
| **llama.cpp** | 效能最佳化，GGUF 原生 | 進階使用者，調參 |
| **MLX** | Apple Silicon 最快 | Mac 使用者 |
| **LM Studio** | GUI，GGUF 即用 | 桌面使用者 |
| **oMLX** | Apple Silicon 專用 server；continuous batching + SSD KV cache tiering + SwiftUI app；`omlx launch claude` 一鍵接 coding agent | Mac 多請求 / 大 context / LLM server 場景 |
| **vLLM / SGLang** | 高吞吐量批次推論 | 伺服器部署，API |
| **LiteRT-LM** | 手機端推論 | Android/iOS 邊緣部署（Gemma 專用） |

```bash
# Ollama 快速上手
ollama run qwen3.5             # 9B（預設，6.6GB）
ollama run qwen3.5:35b         # 35B-A3B MoE（~24GB）
ollama run qwen3.5:27b         # 27B Dense
ollama run gemma4:e4b          # Gemma 4 E4B（9.6GB）
ollama run gemma4:26b-a4b      # Gemma 4 MoE（~18GB）
ollama run gemma4:31b          # Gemma 4 31B Dense（~20GB）
```

---

## Gemma 4 vs Qwen 3.5 選型決策樹

```
我的主要用途？
├── 中文任務 → Qwen 3.5（毫無爭議）
├── Agent / Tool Use → Qwen 3.5（TAU2-Bench 壓倒性優勢）
├── 長文件 RAG → Qwen 3.5（KV cache 效率，190K+ context）
├── 競賽程式設計 → Gemma 4 31B（Codeforces ELO 2150）
├── 手機 / 邊緣部署 → Gemma 4 E2B/E4B（LiteRT-LM + 音訊）
└── 非中文多語言 → Gemma 4（MMMLU 88.4%）

我的硬體？
├── ≤16GB VRAM → Qwen 3.5 9B Q4 (6.6GB) 或 35B-A3B APEX Mini
├── 24GB VRAM (RTX 4090) → Qwen 3.5 35B-A3B Q4 ⭐ 最佳甜蜜點
└── Apple Silicon 64GB → Qwen 3.5 35B-A3B MLX Q4，60–70 tok/s
```

---

---

## 知識萃取任務 Benchmark（2026-04-09）

> 平台：NVIDIA DGX Spark，vLLM，NVFP4；100 篇產業文本，Ground Truth 由 Claude Sonnet 4.6 生成

### 結果摘要

| 模型 | Entity F1 | Precision | Recall | 速度(s/篇) |
|------|-----------|-----------|--------|-----------|
| Qwen 122B-A10B | **79.9%** | 86.9% | 73.9% | 20.0 |
| **Qwen 35B-A3B** | **77.6%** | 80.3% | **75.1%** | 13.6 |
| Gemma 31B Dense | 73.0% | **95.5%** | 59.1% | 71.2 |
| Gemma 26B-A4B | 65.4% | 93.8% | 50.2% | **4.8** |

### 知識圖譜生產選型結論

**推薦 Qwen 3.5 35B-A3B 或 122B-A10B**，原因：
- 知識萃取任務中 **Recall 比 Precision 更重要**——漏掉的知識無法補救，幻覺可靠 Verifier Agent 事後過濾
- Gemma 31B 的高 Precision（95.5%）/ 低 Recall（59.1%）在此場景是弱點，不是優勢
- Qwen 35B 性價比最佳：F1 只比 122B 低 2.3%，速度快 47%
- Thinking Mode 不划算：慢 20x、token 多 30x，反而因截斷降低 parse 成功率
- Gemma 26B × 3 次（F1 71.4%）仍輸 Qwen 122B × 1 次（79.9%）

---

---

## 硬體 Prefill 效能比較（含 Apple M5 Pro 實測）

> Doro 實測（2026-06-06）；其餘欄位為外部 benchmark（llm-tracker.info、GitHub）

### Prefill 跨硬體比較表

| 硬體 | 模型 | Prompt | Prefill tok/s | 來源 |
|------|------|--------|--------------|------|
| M5 Pro 48GB | Qwen3.6-35B-A3B | 7K token | **1,212** | Doro 實測 2026-06-06 |
| M5 Max 128GB | Qwen3.5-9B 4-bit | 22K token | 2,614 | GitHub benchmark |
| DGX Spark (GB10) | GPT-OSS 120B | 128K | ~1,723 | 未獨立驗證，120B 規模 |
| Strix Halo (AI Max+ 395) | Qwen3-30B-A3B 4-bit | pp512 | 119 | llm-tracker.info |

**關鍵洞見：M5 Pro prefill 約為 Strix Halo 的 10 倍**（同為 MoE 模型級別，7K token compute-bound 條件下）。差距根本原因是軟體生態：MLX 在 Apple Silicon 的 GPU 利用率遠優於 AMD APU 上的 ROCm / Vulkan。

**注意**：DGX Spark 的 ~1,723 tok/s 對應 120B 規模模型，M5 Pro 的 1,212 tok/s 對應 35B MoE，兩者規模不同，不宜直接比較；列於此表僅供數量級參考。

---

## Strix Halo vs DGX Spark 硬體比較（2026-06-06）

> 測試模型：GPT-OSS 120B，128k context，llama.cpp

### 規格對照

| 項目 | Strix Halo（AMD AI Max+ 395） | DGX Spark（NVIDIA GB10） |
|------|-------------------------------|--------------------------|
| 記憶體 | 128GB LPDDR5X | 128GB LPDDR5X |
| **記憶體頻寬** | **~256 GB/s** | **273 GB/s** |
| AI 算力 | 126 TOPS INT8（系統） | ~1000 TFLOPS FP4（1 PFLOP） |
| 計算單元 | Radeon 8060S iGPU (40 CU) + XDNA2 NPU | Blackwell GPU + 20 核 Arm CPU |
| 軟體生態 | ROCm / Vulkan | CUDA（成熟） |
| 作業系統 | x86，Windows/Linux，可遊戲 | DGX OS（ARM），開發專用 |
| 價格（約） | $1,500–2,400 | $3,999–4,699 |

### 實測效能（Strix Halo vs DGX Spark，GPT-OSS 120B）

| 平台 | Prefill（tok/s） | Decode（tok/s） |
|------|-----------------|-----------------|
| DGX Spark | ~1,723 | ~38.6 |
| Strix Halo | ~340 | ~34.1 |
| **倍率** | **~5× 快** | **~1.1×（幾乎相同）** |

**核心結論**：兩台「吐字速度」同級（都被 ~256–273 GB/s 頻寬鎖死）；DGX Spark 的價值在 prefill / 訓練 / 影像生成 / CUDA 生態，不是讓聊天生成變快。

### TOPS 數字解讀（以 Strix Halo 為例）

- 126 TOPS INT8 = 系統全域合計（NPU + iGPU + CPU），無法合併使用，是行銷加總
- 50 TOPS = XDNA2 NPU（LM Studio **未使用**）
- 真正跑 LLM 的是 iGPU，且 LLM 用 FP16 ≈ INT8 算力的一半
- **最有感的 tok/s 看頻寬，不是 TOPS**
