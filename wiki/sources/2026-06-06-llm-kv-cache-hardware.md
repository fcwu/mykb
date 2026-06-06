---
title: 本地 LLM × KV Cache × 硬體：LM Studio、Strix Halo vs DGX Spark vs Apple M5 Pro
date: 2026-06-06
category: ai-agent
description: KV cache 原理、Unified KV Cache、head_dim 差異、Prefill vs Decode 根本差異，以及 Strix Halo、DGX Spark、Apple M5 Pro 的跨平台 prefill 實測比較。
tags: [ai, llm, local-inference, kv-cache, hardware, amd, nvidia, apple-silicon, strix-halo, dgx-spark, m5-pro]
---

# 本地 LLM × KV Cache × 硬體：LM Studio、Strix Halo vs DGX Spark vs Apple M5 Pro

> **來源**：整理自技術對話（2026-06-06）；Apple M5 Pro 數據為 Doro 實測，Strix Halo 數據來自 [llm-tracker.info](https://llm-tracker.info/AMD-Strix-Halo-(Ryzen-AI-Max+-395)-GPU-Performance)，M5 Max 數據來自 [GitHub benchmark](https://github.com/itsmostafa/inference-speed-tests)

## 核心觀點

### LM Studio Unified KV Cache（軟體功能，非硬體）

- **Unified KV Cache** 是 LM Studio 的推論引擎設定：並行請求共用同一個 KV cache 池，不是每個請求各切固定一塊。**與 AMD 統一記憶體硬體架構無關**，只是同名巧合。
- KV cache 在**載入模型當下**就依 `context_length` 一次預配（preallocate）完成，對話長了不會讓 VRAM 慢慢爬升；設越長的 context → 載入瞬間就吃越多。
- **Unified 開啟 vs 關閉的差異**：
  - 開啟：共享池，`n_ctx × 1`，省記憶體（省 n_seq_max 倍），並行時可能搶，用滿可能 crash
  - 關閉：各 slot 獨立，`n_ctx × n_seq_max`（預設 4），穩定但浪費
  - **單人使用建議保持開啟**

### KV Cache 原理

- **作用**：存過去 token 的 K/V 向量，生成下一個 token 時不必重算整段序列，是「用記憶體換速度」
- **大小公式**：`2 × 層數 × context_length × KV_heads × head_dim × bytes_per_element`
- **MoE 不省 KV cache**：attention 層是 dense、全部要算；KV cache 只跟 attention 結構有關，跟 active 參數量無關

### 為什麼 Gemma 4 比 Qwen MoE 更吃 VRAM

- 關鍵在 **head_dim**，不是 dense vs MoE：
  - Gemma 4：sliding 層 head_dim=256，global 層 head_dim=512
  - Qwen：head_dim=128
  - 每個 token 的 K/V 比 Qwen 重 2~4 倍；global 層隨 context 線性成長
- **Flash Attention（FA2）**：把 attention 切塊在 GPU 高速 SRAM 邊算邊累加，不存完整 N×N 矩陣 → 省記憶體 + 更快。FA2 head_dim 上限是 256，Gemma 4 的 global 層 head_dim=512 需特別處理。

### GTT（Graphics Translation Table）— AMD APU 專屬

- 在統一記憶體 APU 上，讓 iGPU 把系統 RAM 當 VRAM 用的機制。權重 + KV cache 都從這個池分配。
- Windows：調 BIOS Variable Graphics Memory（VGM）
- Linux：調 `amdttm.pages_limit`（舊參數名：`amdgpu.gttsize`）

### Prefill vs Decode：兩階段的根本差異

| 階段 | 性質 | 瓶頸 | 指標 |
|------|------|------|------|
| Prefill（處理 prompt） | Compute-bound | 算力 | TOPS / FLOPS |
| Decode（逐字生成） | Bandwidth-bound | 記憶體頻寬 | GB/s → tok/s |

- **TTFT（Time To First Token）≈ prefill 時間 ≈ prompt 長度 ÷ prefill 速度**，是算力問題，不是頻寬問題
- **TOPS 注意事項**：`TOPS = MAC 數量 × 2 × 時脈`；一定要看精度（INT8/INT4/FP16 不同），是理論峰值

### Strix Halo（AMD Ryzen AI Max+ 395）的 TOPS 拆解

- 126 TOPS INT8 = 系統全域合計（NPU + iGPU + CPU）
- 50 TOPS = XDNA2 NPU（LM Studio 未使用）
- 76 TOPS ≈ Radeon 8060S iGPU + CPU
- **LM Studio 跑 LLM 實際看的是 iGPU 頻寬，而非 TOPS**；LLM 用 FP16 ≈ INT8 的一半算力

## 關鍵洞見

### Strix Halo vs DGX Spark 規格對照

| 項目 | Strix Halo (AMD AI Max+ 395) | DGX Spark (GB10) |
|------|-------------------------------|-------------------|
| 記憶體 | 128GB LPDDR5X | 128GB LPDDR5X |
| **記憶體頻寬** | **~256 GB/s** | **273 GB/s** |
| AI 算力 | 126 TOPS INT8（系統） | ~1000 TFLOPS FP4（1 PFLOP） |
| 計算單元 | Radeon 8060S iGPU (40 CU) + XDNA2 NPU | Blackwell GPU + 20 核 Arm CPU |
| 軟體生態 | ROCm / Vulkan | CUDA（成熟） |
| 作業系統 | x86，正常 Windows/Linux，可遊戲 | DGX OS（ARM），開發專用 |
| 價格（約） | $1,500–2,400 | $3,999–4,699 |

### 實測效能對照（GPT-OSS 120B, 128k context, llama.cpp）

| 平台 | Prefill（tok/s） | Decode（tok/s） |
|------|-----------------|-----------------|
| DGX Spark | ~1,723 | ~38.6 |
| Strix Halo | ~340 | ~34.1 |
| **倍率** | **~5× 快** | **~1.1×（幾乎相同）** |

**結論**：兩台「吐字速度」同級（都被 ~256–273 GB/s 頻寬鎖死）；DGX Spark 的價值在 prefill / 訓練 / 影像生成 / CUDA 生態，不是讓聊天生成變快。

### 跨平台 Prefill 比較（含 Apple M5 Pro 實測）

> Doro 實測：M5 Pro 48GB × Qwen3.6-35B-A3B × 7K token prompt，2026-06-06

| 硬體 | 模型 | Prompt | Prefill tok/s | 來源 |
|------|------|--------|--------------|------|
| M5 Pro 48GB | Qwen3.6-35B-A3B | 7K token | **1,212** | Doro 實測 2026-06-06 |
| M5 Max 128GB | Qwen3.5-9B 4-bit | 22K token | 2,614 | GitHub benchmark |
| DGX Spark (GB10) | GPT-OSS 120B | 128K | ~1,723 | 未獨立驗證，120B 規模 |
| Strix Halo (AI Max+ 395) | Qwen3-30B-A3B 4-bit | pp512 | 119 | llm-tracker.info |

**M5 Pro vs Strix Halo（同為 MoE 30–35B 等級）**：M5 Pro 的 1,212 tok/s 約為 Strix Halo 的 **10 倍**，根本原因是 MLX 在 Apple Silicon 的 GPU 利用率遠優於 AMD APU 上的 ROCm / Vulkan。

> 注意：DGX Spark 的 ~1,723 tok/s 是 120B 規模模型，M5 Pro 的 1,212 tok/s 是 35B MoE，規模不同，不宜直接比較；列於此表供數量級參考。

### 四個心智模型

1. **第一個字等多久 = 算力問題；之後吐字多快 = 頻寬問題。**
2. **KV cache 在載入時就定案**（由 context length 決定），不是用著用著才長。
3. **MoE 救 decode、不救 prefill、不省 KV cache。**
4. **head_dim 是 KV cache 的隱藏放大器**：Gemma 4 大、Qwen 小，所以小模型不一定省 VRAM。

## 原文重點段落

> KV cache 在「載入模型當下」就依 `context_length` 一次預先配置 (preallocate) 完成，不會隨對話變長才爬升。設越長 → 載入瞬間就吃越多。

> 真正讓 Gemma 4 爆 VRAM 的是它超大的 head_dim（sliding 層 256 / global 層 512），每個 token 的 K/V 比 head_dim=128 的 Qwen 重 2~4 倍，且 global 層隨 context 線性成長。

> 兩台「吐字速度」同級（都被 ~256–273 GB/s 頻寬鎖死）；DGX Spark 的價值在 prefill / 訓練 / 影像生成 / CUDA 生態，而不是讓聊天生成變快。
