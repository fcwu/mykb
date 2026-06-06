---
title: oMLX — Apple Silicon LLM 推論伺服器
type: entity
tags: [ai, llm, apple-silicon, mlx, local-inference, kv-cache, swiftui, macos, open-source]
date: 2026-06-06
category: ai-agent
description: oMLX 是專為 Apple Silicon Mac 設計的本地 LLM 推論伺服器，主打 continuous batching、SSD KV cache tiering 與原生 SwiftUI 管理 UI，v0.4.0 完成全面重寫。
---

# oMLX — Apple Silicon LLM 推論伺服器

> **來源**：GitHub — [jundot/omlx · Releases](https://github.com/jundot/omlx/releases)（擷取於 2026-06-06）

> 以 MLX 框架為基礎、專為 Apple Silicon Mac 設計的本地 LLM 推論伺服器，主打 continuous batching + SSD KV cache tiering + 原生 macOS 管理 UI。

## 概述

oMLX（作者：jundot）是開源的 Apple Silicon LLM 推論伺服器，建立在 Apple MLX 框架之上。特色是：

- **Continuous batching**：多請求並行推論，不需等前一個完成
- **SSD KV cache tiering**：超過 RAM 容量的 KV cache 自動 spill 到 SSD，讓長 context / 大模型在記憶體有限的 Mac 上可用
- **原生 macOS app（v0.4.0+）**：SwiftUI 管理介面，處理模型下載、狀態監控、設定等
- **CLI 工具**：`omlx start/stop/restart` 等指令（v0.4.1 起）
- **一鍵接 coding agent**：`omlx launch claude` 把本機 LLM 接入 Claude Code

## 關鍵特性

### KV Cache 管理

| 機制 | 版本引入 | 說明 |
|------|---------|------|
| SSD tiering（基礎） | v0.3.x | KV cache 超量時 spill 到 SSD |
| hot_cache_only | v0.3.7 | 強制全 RAM，零 SSD I/O；RAM 充裕時性能最佳 |
| 非同步 SSD 寫入 | v0.3.8 | cache-on 解碼提速，不阻塞 decode |
| paged_ssd_cache v3 | v0.3.9.dev1 | 新版 paged 架構，支援 DeepSeek V4 |
| TurboQuant | v0.4.0 | 批次 KV-cache 量化壓縮，降低 KV 記憶體佔用 |

### 量化（oQ / ParoQuant / TurboQuant）

- **oQ**（on-the-fly Quantization）：KV cache streaming 量化（float16 → 低精度），v0.3.6 引入；M1/M2 prefill ~20% 提速
- **TurboQuant**：批次壓縮已生成 KV cache，v0.4.0 整合；v0.4.1 修 MLA 模型（DeepSeek 等）相容性
- **ParoQuant**：可插拔量化載入器，v0.3.9.dev2 引入

### 推測解碼

- **DFlash**：block diffusion 演算法推測解碼，v0.3.5 引入；v0.3.9.dev2 擴充支援 Gemma 4

### VLM 支援

- v0.3.5：多圖 prefix cache reuse（快取命中率 14%→76%）
- v0.3.8：VLM continuous batching

### macOS UI 演進

| 時期 | UI 架構 |
|------|---------|
| v0.3.x | PyObjC 選單列 + web 後台 |
| v0.4.0+ | 原生 Swift/SwiftUI macOS app（全新 onboarding/設定/狀態/模型管理） |

## 與其他工具的關係

- **本地推論替代方案**：oMLX 是 Apple Silicon 本地推論的重要選項，補足 Ollama/LM Studio 外的 server 選擇；計算效率接近 LM Studio（MLX backend）
- **Claude Code 整合**：`omlx launch claude` 可一鍵將 oMLX 本機 LLM 接入 Claude Code，實現全地端 coding agent

## 演進紀錄

| 時間 | 版本 | 事件 |
|------|------|------|
| 2026-04-15 | v0.3.5 | DFlash 推測解碼；VLM prefix cache reuse |
| 2026-04-16 | v0.3.6 | float16 oQ；oQ streaming 量化 |
| 2026-04-28 | v0.3.7 | hot_cache_only；preset 系統 |
| 2026-04-30 | v0.3.8 | VLM continuous batching；非同步 SSD 寫入 |
| 2026-05-06 | v0.3.9.dev1 | DeepSeek V4 SSD；paged_ssd_cache v3；omlx launch claude |
| 2026-05-12 | v0.3.9.dev2 | Gemma 4 MTP；ParoQuant |
| 2026-06-02 | v0.4.0 | **SwiftUI 全重寫**（分水嶺）；TurboQuant 整合 |
| 2026-06-03 | v0.4.1 | 記憶體壓力穩定；TurboQuant MLA 修復；CLI |
