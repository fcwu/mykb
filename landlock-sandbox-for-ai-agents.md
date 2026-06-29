---
title: 當 AI agent 要跑不可信指令 — Go 把 Landlock 收進標準庫，與 OpenShell 的檔案牢籠
date: 2026-06-29
category: ai-agent
description: Go 提案 #68595 想讓 os/exec 直接生出被 Landlock 關住的子程序。這跟「AI agent 怎麼安全地跑你給它的指令」是同一個問題——NVIDIA OpenShell 已把同一招做進 agent runtime。本文拆 Landlock 機制、它為何比 mount namespace 牢，以及 BestEffort 靜默降級這個坑。
tags: [ai, agent, sandbox, landlock, lsm, linux, security, least-privilege, golang, openshell]
---

# 當 AI agent 要跑不可信指令：Go 把 Landlock 收進標準庫，與 OpenShell 的檔案牢籠

> **來源**：Go 提案 — [golang/go#68595 · syscall: support process sandboxing using Landlock on Linux](https://github.com/golang/go/issues/68595)（經 Golang Weekly #607）；OpenShell — [NVIDIA/OpenShell · GitHub](https://github.com/NVIDIA/OpenShell)（Apache 2.0，2026-03 GTC 開源）

Go 最近接受了一個提案（#68595），要讓 `os/exec` 直接生出「被 Landlock 關起來的子程序」。大多數人看到大概想：喔，又一個安全用的 flag。

這當然有用，但小編覺得更值得講的是它背後那條線：它跟「**AI agent 怎麼安全地跑你給它的指令**」根本是同一個問題，而 NVIDIA 的 OpenShell 已經把同一招做進了 agent runtime。兩件事擺一起看就清楚了——least privilege 在 Linux 上，正從「要 root 預先配置的麻煩事」變成「應用程式自帶的能力」。

## 1. Go 想解的問題：反覆跑數十萬次不可信指令

動機場景很具體：一個 Go 程式對**不可信的輸入**反覆呼叫外部 `git`，數十萬次。每次 `git` 只需要碰一個暫存目錄，卻繼承了父程序的全部檔案存取權。

要收斂這個，現況下你得自己重造 `os/exec`，或繞過 Go threading 模型去玩 `fork(2)`——兩條路都不好走。提案的做法是直接擴充 `syscall.SysProcAttr`：

```go
cmd.SysProcAttr = &syscall.SysProcAttr{
    UseLandlock:     true,
    LandlockRuleset: fd,
}
```

`fd` 是一組 Landlock ruleset，用外部套件 `github.com/landlock-lsm/go-landlock` 建：

```go
fd, err := landlock.V5.BestEffort().OnlyPaths().Ruleset(
    landlock.ROFiles("/usr"),   // 唯讀
    landlock.RWFiles(tmpDir),   // 可讀寫
)
```

三個設計重點：

- **只限制子程序**，父程序不受影響——正是「父程序當 orchestrator、子程序當 untrusted worker」的形狀。
- **向後相容**：沒設就沒事，對既有程式零影響。
- `no_new_privs` 與 Landlock 設定**正交**，各自獨立開關。

值得一提：這提案是更早 #47049（已撤回）的續命版。標準庫為它試了兩次，本身就是訊號——Landlock 已被當成 Linux 上做 least privilege 的標準途徑，不再是 niche 技巧。

## 2. Landlock 是什麼：免 root 的「程序自己把自己關起來」

一句話定位：**Landlock 是讓非特權程序自我設限的 Linux 核心機制**——程序宣告一組規則（哪些路徑唯讀、哪些可讀寫，其餘預設拒絕），套上之後連自己都解不開。

運作分兩階段：`landlock_create_ruleset` 開一張空 ruleset → `landlock_add_rule` 逐條加「此路徑可讀」「此目錄可寫」→ `landlock_restrict_self` 套到當前 thread，**之後就回不去了**。沒有「解除沙箱」的 API，這是刻意的。

它跟傳統幾招（chroot、mount namespace、SELinux/AppArmor policy）的差別在三點：

1. **免 root**。chroot 要 root、AppArmor policy 要管理員預寫；Landlock 不用——任何一般使用者程序自己呼叫上面三個 syscall，就能把自己關進更小的牢籠。安全收斂變成應用程式自帶的能力。
2. **不可逆，且不理 capability**。`restrict_self()` 之後規則只能愈來愈嚴，**即使後續拿到 `CAP_SYS_ADMIN` 也逃不出去**。這是它的殺手鐧，下一節會看到為什麼。
3. **以核心物件為單位**。早期只授權檔案路徑（`ROFiles` / `RWFiles` / `RODirs` / `RWDirs`），沒列到的一律拒絕——是 allowlist，不是 blocklist。

### ABI 版本：Landlock 早就不只管檔案

第 1 節那行 `landlock.V5` 的 `V5` 不是套件版本號，是 **Landlock 的 ABI 版本**。Landlock 自 kernel 5.13 起以 ABI 逐步長出新能力，每一版對應夠新的 kernel：

| ABI | Kernel | 加了什麼 |
|---|---|---|
| v1 | 5.13 | 檔案系統存取控制（讀／寫／執行／刪除，路徑級） |
| v2 | 5.19 | `LANDLOCK_ACCESS_FS_REFER`——跨目錄 rename / link |
| v3 | 6.2 | `truncate` 操作 |
| v4 | 6.7 | **TCP `bind` / `connect` 限制**（指定允許的 port）——開始管網路 |
| v5 | 6.10 | 對字元／區塊裝置的 `ioctl` 限制 |

重點：**v4 起 Landlock 已不只是「檔案沙箱」**，能限制程序只准 connect 哪些 TCP port。所以 `landlock.V5` 要的是 kernel 6.10 等級的完整能力；跑在舊 kernel 上，網路與 ioctl 規則會被 `BestEffort()` 默默丟掉（第 4 節）。

### 跟 seccomp 互補，不是二選一

**seccomp 過濾 syscall 編號與參數**（「禁 `mount`」「禁 `ptrace`」），看的是「你呼叫了哪個 syscall」；**Landlock 看的是「你想碰哪個路徑、哪個 port」**，是 path-aware 的。seccomp 表達不了「能 `open`，但只能 open 這個目錄」，Landlock 正好補上。兩者疊用——seccomp 收 syscall 面，Landlock 收檔案與網路物件面。下一節的 OpenShell 正是這樣疊的。

## 3. 同一個問題，換到 AI agent：OpenShell

一個 autonomous agent 要做事就得跑指令、讀寫檔案、連網路，可你並不完全信任它下一步幹嘛。這跟「反覆跑不可信 `git`」是**同一個問題**，只是不可信的一方從外部 repo 換成模型輸出。

NVIDIA 的 OpenShell（2026-03 GTC 以 Apache 2.0 開源）正衝這個來：官方定位它為「autonomous AI agent 的安全、私密 runtime」，用核心級隔離加宣告式 YAML policy，讓 agent 在拿不到完整檔案、憑證、外部網路的前提下做事。sandbox 預設開檔案、網路、process 三種隔離，底層疊三層核心機制：

- **seccomp**——過濾 syscall，擋掉 `ptrace`、`mount`、raw socket。
- **Landlock LSM**——把檔案存取鎖在 policy 的 `read_only` / `read_write` 目錄，沒列到的碰不到。**這層跟 Go 提案是同一個機制**。
- **network namespace**——隔離流量。

### policy：static 鎖死，dynamic 熱重載

OpenShell 的 policy 是一份 YAML，頂層欄位大致為 `version`、`filesystem_policy`、`landlock`、`process`、`network_policies`、`inference`，分兩種生命週期——這個切分就是設計重點：

- **靜態段（filesystem、process）建立時鎖死**：agent 跑起來後不能再放寬自己能碰的檔案。路徑須絕對、不准 `..`，server 會正規化路徑、擋掉 traversal 逃逸。
- **動態段（network、inference）對著正在跑的 sandbox 熱重載**（`openshell policy set`，`--wait` 會擋到新 policy 確認載入），不必重建容器。

兩個 agent 場景的關鍵段：

**network policy** 不是「准／不准上網」，而是宣告 endpoint 配 binary——**只有列出的 binary 能連列出的 endpoint**，例如「只准 `curl` 連 `api.github.com`」。對 agent 防資料外送（exfiltration）顆粒度剛好。

**inference policy** 更有意思：**把模型 API 呼叫重導到受控後端**。agent 以為在打某個 LLM endpoint，其實被導到你控制的 proxy——要審計它送了什麼 prompt、收了什麼，這是很乾淨的切入點。

compute 後端 Docker、Podman、MicroVM、Kubernetes 都收，同一份 policy 能從筆電帶到 GPU cloud。

### 為什麼檔案那層用 Landlock，而不是 mount namespace

agent sandbox 為了能做事，常帶著 `SYS_ADMIN` 這種較高 capability。用 mount namespace 當牢籠的問題是——同一個 `SYS_ADMIN` 能再 `mount` / `pivot_root` 逃出去，鑰匙等於掛在犯人手上。Landlock 的 `restrict_self` 不理 capability，這正是第 2 節「不可逆、不理 capability」發光之處：agent 就算升權，檔案牢籠照樣關死。用對工具，差別在這。

### 調 policy 的實際手感

官方建議的調法也務實：先跑 sandbox、用 `openshell term` 即時看哪些連線被擋，再熱重載收緊或放寬，不必重建。這種「邊看 deny log 邊收緊」的迴圈，比一次寫對一大份 policy 實際——先擋再補，遠比一開始開太鬆安全。

## 4. 一個會讓你以為有沙箱、其實沒有的坑

回來算 `BestEffort()` 那筆帳。

Landlock 不是「kernel 夠新就一定有」。它是**核心編譯選項加開機 LSM 設定**（`CONFIG_SECURITY_LANDLOCK` 加上 `lsm=` 開機參數排進去）。所以你可能跑在 kernel 5.13+ 的系統上，LSM stack 裡卻**根本沒有 landlock**——一些嵌入式、NAS 級或廠商客製 build 就這樣，stack 只有 `capability,apparmor`。

這時 `BestEffort()` 的「靜默降級」從貼心變陷阱：檔案隔離**悄悄退回只剩 DAC**（Discretionary Access Control，傳統 Unix rwx），程式照跑、零 error，但那道檔案牆根本沒立起來。一個 world-writable 或 agent 自己擁有的路徑，DAC 不會幫你擋。

教訓很直接：**`BestEffort()` 之後要驗證實際生效強度，別假設它套上了。** OpenShell 這類 runtime 會在啟動時把「Landlock 不可用」印成高嚴重度 finding，就是要讓你知道少了哪一層，而不是默默假裝它在。

## 5. 小編的看法

小編認為真正的故事是：**Landlock 正從 security 圈的 niche 機制，變成基礎建設的預設選項**。兩個獨立方向同時指向它——語言層（Go 做進 `os/exec`）與 agent runtime 層（OpenShell 當 filesystem 第一道防禦）。兩邊不約而同選它，比任何單一專案的背書都有說服力。

但有個 trade-off 要講清楚：**least privilege 不該把賭注全押在單一層**。OpenShell 疊三層（syscall、檔案、網路）不是為了好看——是為了某一層失效時其他層還在。第 4 節正好示範：平台沒有 Landlock 時，檔案隔離蒸發，撐住場面的是 seccomp 與 network namespace。所以務實的問法不是「有沒有沙箱」，而是「**每一層各擋掉什麼、哪一層在這平台上其實是空的**」。

若目標平台連 Landlock 都沒有、檔案隔離又非要不可，microVM（OpenShell 支援的後端之一）是比 namespace 更乾淨的退路——用硬體虛擬化邊界補上那層拿不到的核心機制。

收束：這整件事是 defense-in-depth 老原則的具體演出——把最小權限從運維手動配置，往下推到應用程式自帶、再推到語言標準庫。方向小編很看好；但落地時，「靜默降級」是會讓你誤判自己安全的地雷，部署前務必實測每一層是否真的生效。

### 參考連結

- [golang/go#68595 · syscall: support process sandboxing using Landlock on Linux](https://github.com/golang/go/issues/68595)（前身 [#47049](https://github.com/golang/go/issues/47049)，已撤回）
- [github.com/landlock-lsm/go-landlock](https://github.com/landlock-lsm/go-landlock) — 建立 ruleset 的 Go 套件
- [NVIDIA/OpenShell · GitHub](https://github.com/NVIDIA/OpenShell) — AI agent 安全 runtime（Apache 2.0）
- [Landlock 官方文件](https://docs.kernel.org/userspace-api/landlock.html) — kernel.org 的 unprivileged sandboxing 說明
