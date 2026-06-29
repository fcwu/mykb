---
title: 當 AI agent 要跑不可信指令 — Go 把 Landlock 收進標準庫，與 OpenShell 的檔案牢籠
date: 2026-06-29
category: ai-agent
description: Go 提案 #68595 想讓 os/exec 直接生出被 Landlock 關住的子程序。這跟「AI agent 怎麼安全地跑你給它的指令」其實是同一個問題——NVIDIA OpenShell 已把同一招做進 agent runtime。本文拆 Landlock 機制、它為何比 mount namespace 牢，以及 BestEffort 靜默降級這個容易被忽略的坑。
tags: [ai, agent, sandbox, landlock, lsm, linux, security, least-privilege, golang, openshell]
---

# 當 AI agent 要跑不可信指令：Go 把 Landlock 收進標準庫，與 OpenShell 的檔案牢籠

> **來源**：Go 提案 — [golang/go#68595 · syscall: support process sandboxing using Landlock on Linux](https://github.com/golang/go/issues/68595)（經 Golang Weekly #607）；OpenShell — [NVIDIA/OpenShell · GitHub](https://github.com/NVIDIA/OpenShell)（Apache 2.0，2026-03 GTC 開源）

Go 最近接受了一個提案（#68595），要讓 `os/exec` 直接生出「被 Landlock 關起來的子程序」。大多數人看到大概想：喔，又一個安全用的 flag。

這當然有用，但我覺得更值得講的，是它背後那條線——它跟「**AI agent 怎麼安全地跑你給它的指令**」根本是同一個問題。而 NVIDIA 的 OpenShell 已經把同一招做進了 agent runtime。把這兩件事擺在一起看，會比單獨讀任何一篇 release note 都清楚：least privilege 在 Linux 上，正在從一個要 root 預先配置的麻煩事，變成應用程式自己就能上的內建能力。

## 1. Go 想解的問題：一個 binary 反覆跑數十萬次不可信指令

提案的動機場景很具體：你有一個 Go 程式，會對**不可信的輸入**反覆呼叫外部 `git`——數十萬次。每次 `git` 只需要碰一個暫存目錄，不需要看到你整個檔案系統。但 `os/exec` 跑出來的子程序，預設繼承父程序的全部檔案存取權。

現況下要收斂這個，你得自己重造一套 `os/exec`，或者繞過 Go 的 threading 模型去玩 `fork(2)`——兩條路都不好走。所以提案的做法是直接擴充 `syscall.SysProcAttr`：

```go
cmd.SysProcAttr = &syscall.SysProcAttr{
    UseLandlock:     true,
    LandlockRuleset: fd,
}
```

那個 `fd` 是一組 Landlock ruleset，用外部套件 `github.com/landlock-lsm/go-landlock` 建：

```go
fd, err := landlock.V5.BestEffort().OnlyPaths().Ruleset(
    landlock.ROFiles("/usr"),   // 唯讀
    landlock.RWFiles(tmpDir),   // 可讀寫
)
```

三個設計重點，每一個都很關鍵：

- **只限制子程序**，父程序完全不受影響——這正是「父程序當 orchestrator、子程序當 untrusted worker」的形狀。
- **向後相容**：對既有程式零影響，沒設就是沒事。
- `no_new_privs` 與 Landlock 設定**正交**——兩個各自獨立開關，別把它們綁在一起想。

值得停一下的是：這個提案其實是更早 #47049 的續命版（前者被撤回了）。連 Go 標準庫都覺得這值得內建、而且試了兩次，本身就是一個訊號——Landlock 已經被當成 Linux 上做 least privilege 的標準途徑，不再是 niche 技巧。

## 2. Landlock 到底是什麼：一個免 root 的「程序自己把自己關起來」

一句話抓定位：**Landlock 是一個讓非特權程序自我設限的 Linux 核心機制**——程序宣告一組規則（哪些路徑唯讀、哪些可讀寫，其餘預設拒絕），套上去之後連自己都解不開。

它跟傳統那幾招（chroot、mount namespace、SELinux/AppArmor policy）最大的差別是三件事：

1. **免 root**。chroot 要 root，AppArmor policy 要管理員預先寫好。Landlock 不用——任何一般使用者程序都能呼叫 `landlock_create_ruleset` / `landlock_add_rule` / `landlock_restrict_self`，自己把自己關進更小的牢籠。安全收斂變成應用程式自帶的能力，不靠運維配置。
2. **不可逆，而且不理 capability**。`restrict_self()` 之後規則只能愈來愈嚴，**即使後續拿到 `CAP_SYS_ADMIN` 也逃不出去**。這點是它的殺手鐧，下一節會看到為什麼。
3. **以路徑為單位**。`ROFiles` / `RWFiles` / `RODirs` / `RWDirs` 授權，沒列到的路徑一律拒絕。心智模型是 allowlist，不是 blocklist。

Landlock 自 kernel 5.13 起以 **ABI 版本**演進，後續版本陸續加上網路、`ioctl`、更細的權限。`BestEffort()` 就是用來吃這個版本差異的——在不支援新規則的舊 kernel 上，它會**自動 drop 套不上的規則繼續跑，而不是報錯**。這個設計很貼心，但它埋了一個坑，第 4 節會回來算這筆帳。

## 3. 同一個問題，換到 AI agent 世界：OpenShell

把鏡頭轉到 agent。一個 autonomous agent 要做事，就得跑指令、讀寫檔案、連網路——可是你並不完全信任它接下來會幹嘛。這跟「反覆跑不可信 `git`」是**字面上同一個問題**，只是不可信的那一方從外部 repo 換成了模型的輸出。

NVIDIA 的 OpenShell（2026 年 3 月在 GTC 以 Apache 2.0 開源）就是衝這個來的：一個專門跑 autonomous AI agent 的安全 runtime，用核心級隔離搭配宣告式 YAML policy，讓 agent 在拿不到你完整檔案、憑證、外部網路的前提下做事。它的防禦疊了三層核心機制：

- **seccomp**——過濾 syscall，擋掉 `ptrace`、`mount`、raw socket 這類危險呼叫。
- **Landlock LSM**——把 agent 的檔案存取鎖在 policy 裡 `allowed_reads` / `allowed_writes` 宣告的目錄。**就是這一層，跟 Go 提案是同一個機制**。
- **network namespace**——隔離流量。

policy 是宣告式 YAML：靜態段（filesystem、process）在 sandbox 建立時就鎖死，動態段（network、inference）可以對著一個在跑的 sandbox 熱重載。compute 後端則 Docker、Podman、MicroVM、Kubernetes 都收。

這裡有一個容易被滑過、但其實很漂亮的設計決定：**為什麼檔案那層要用 Landlock，而不是用 mount namespace 把 agent chroot 進一個小目錄就好？**

因為 agent sandbox 為了能做事，容器內常常帶著像 `SYS_ADMIN` 這種較高的 capability。而 mount namespace 當牢籠的問題是——同一個 `SYS_ADMIN` 可以再 `mount` / `pivot_root` 把自己搬出去。牢籠的鑰匙就掛在犯人手上。Landlock 的 `restrict_self` 不理 capability，這就是第 2 節那個「不可逆、不理 capability」性質真正發光的地方：就算 agent 後來升權，檔案牢籠依然關得死死的。用對工具，差別在這。

## 4. 一個會讓你以為有沙箱、其實沒有的坑

現在回來算 `BestEffort()` 那筆帳。

Landlock 不是「kernel 夠新就一定有」。它是一個**核心編譯選項加開機時的 LSM 設定**（`CONFIG_SECURITY_LANDLOCK` 加上 `lsm=` 開機參數要把它排進去）。意思是：你完全可能跑在一個 kernel 5.13+、看起來夠新的系統上，但它的 LSM stack 裡**根本沒有 landlock**——一些嵌入式、NAS 級或廠商客製的 Linux build 就是這樣，他們的 stack 可能只有 `capability,apparmor`，Landlock 沒被編進去。

這時候 `BestEffort()` 的「靜默降級」就從貼心變成陷阱：你的檔案隔離那一層會**悄悄退回只剩 DAC**（Discretionary Access Control，就是傳統 Unix 的 rwx 權限），程式照跑、沒有任何 error，但你以為有的那道檔案牆其實沒立起來。一個 world-writable 或 agent 自己擁有的路徑，DAC 根本不會幫你擋。

教訓很直接：**`BestEffort()` 之後，要去驗證實際生效的強度，不能假設它套上了。** OpenShell 這類 runtime 會在啟動時把「Landlock 不可用」當成一條高嚴重度的 finding 印出來，就是這個道理——讓你知道你少了哪一層，而不是默默假裝它在。

## 5. 我的看法

我認為這裡真正的故事，是 **Landlock 正在從一個 security 圈的 niche 機制，變成基礎建設的預設選項**。判斷依據有兩個獨立的方向同時指向它：語言層（Go 想把它做進 `os/exec`）和 agent runtime 層（OpenShell 把它當 filesystem 防禦的第一道）。兩邊都不約而同選了它，這比任何單一專案的背書都有說服力。

但有個務實的 trade-off 要講清楚：**least privilege 不該把賭注全押在單一層上**。OpenShell 疊三層（syscall、檔案、網路）不是為了好看——它是為了在任何一層失效時，其他層還在。第 4 節那個坑正好示範了為什麼：當底層平台沒有 Landlock，你的檔案隔離就蒸發了，這時候撐住場面的是 seccomp 跟 network namespace。所以設計一個會跑不可信工作的系統時，務實的問法不是「我有沒有沙箱」，而是「**我每一層各擋掉什麼、哪一層在這個平台上其實是空的**」。

如果你的目標平台連 Landlock 都沒有、而檔案隔離又非要不可，那 microVM（OpenShell 也支援的後端之一）會是比 namespace 更乾淨的退路——用硬體虛擬化邊界換掉那層你拿不到的核心機制。

收束一下：這整件事是 defense-in-depth 這個老原則的一次具體演出——把「最小權限」從運維手動配置往下推到應用程式自帶、再往下推到語言標準庫。方向我很看好；但落地時，「靜默降級」是那個會讓你誤判自己很安全的地雷，部署前一定要實測每一層到底有沒有生效。

### 參考連結

- [golang/go#68595 · syscall: support process sandboxing using Landlock on Linux](https://github.com/golang/go/issues/68595)（前身 [#47049](https://github.com/golang/go/issues/47049)，已撤回）
- [github.com/landlock-lsm/go-landlock](https://github.com/landlock-lsm/go-landlock) — 建立 ruleset 的 Go 套件
- [NVIDIA/OpenShell · GitHub](https://github.com/NVIDIA/OpenShell) — AI agent 安全 runtime（Apache 2.0）
- [Landlock 官方文件](https://docs.kernel.org/userspace-api/landlock.html) — kernel.org 的 unprivileged sandboxing 機制說明
