---
title: 把失控的 AI agent 關進牢籠 — OpenShell 的六層 kernel 沙箱，與 Landlock 這塊地基
date: 2026-06-29
category: ai-agent
description: NVIDIA OpenShell 把任意 agent / CLI 關進 kernel 級隔離 + 宣告式 policy 的沙箱裡跑。本文讀它的原始碼，深拆 Gateway/Supervisor 架構、實際疊起來的六層 kernel 沙箱、為何檔案隔離選 Landlock 而非 mount namespace、inference router 怎麼讓 agent 拿不到模型 API key、Z3 prover 怎麼守 policy；並用 Go 提案 #68595 佐證 Landlock 正在標準化。
tags: [ai, agent, sandbox, openshell, nvidia, landlock, seccomp, nftables, libkrun, microvm, security, least-privilege, golang]
---

# 把失控的 AI agent 關進牢籠：OpenShell 的六層 kernel 沙箱，與 Landlock 這塊地基

> **來源**：OpenShell — [NVIDIA/OpenShell · GitHub](https://github.com/NVIDIA/OpenShell)（Apache-2.0，2026-03 GTC 開源）＋[官方 docs](https://docs.nvidia.com/openshell/about/overview)；Go 提案 — [golang/go#68595](https://github.com/golang/go/issues/68595)（經 Golang Weekly #607）

一個 autonomous coding agent（Claude Code、Codex、Copilot CLI…）本質上是「會自己決定要跑什麼指令、連什麼網路」的程式。你直接在開發機或 server 上放它跑，攻擊面就攤開了：它能把 source code POST 到任意端點、能被 prompt injection 拐走環境變數裡的 API key、能改到 `/usr`、能連 `169.254.169.254` 偷雲端 IAM 憑證。

業界對這個問題有兩個方向的動作。一個在**語言層**：Go 接受了提案 #68595，要讓 `os/exec` 直接生出「被 Landlock 關起來的子程序」。另一個在 **runtime 層**：NVIDIA 開源了 OpenShell，把整個 agent 關進 kernel 級隔離的沙箱。小編覺得這兩件事擺一起看最有意思——它們指向同一個底層機制（Landlock），也指向同一個原則（least privilege 該下沉成基礎設施自帶的能力）。這篇主要拆 OpenShell 與 Landlock，Go 提案放在後面當佐證。

## 1. OpenShell 是什麼：給 AI agent 的 K8s + Istio + Vault

OpenShell 是 NVIDIA 開源（Apache-2.0）的「autonomous AI agent 安全 runtime」。一句話：**把任意 agent / CLI 關進有 kernel 級隔離與宣告式 policy 的沙箱裡跑，限制它能碰的檔案、網路、process 與推理後端，讓「被騙或失控的 agent」做不了壞事。**

它的核心立場很明確——**把安全 policy 放在 agent 構不到的基礎設施層**，跟 application 層的 agent 操作分離。policy 在系統層強制執行，agent 改不了自己的牢籠。三大賣點：

- **沒有 opt-out 的沙箱**：agent 第一天就鎖在裡面，無法逃逸、沒有解鎖開關。
- **default-deny egress**：每條對外連線都得通過 policy proxy，預設全擋。
- **把 LLM 憑證從 agent 手上拿走**：模型 API 呼叫被攔截改道，連 agent 自己都拿不到金鑰。

一個好記的定位：**「給 AI agent 的 Kubernetes + Istio + Vault」**，但收斂成單一 control plane 加每沙箱一個 in-process 安全邊界。

> ⚠️ **成熟度是最大的 caveat**。官方 README 自稱 **alpha / "single-player mode"**（proof-of-life：one developer、one environment、one gateway），**多租戶還在做、尚未到位**。2026-03 才開源、約 6 個月新、NVIDIA（Ali Golshan 團隊）主導；ServiceNow Knowledge 2026 上發表、Project Arc 拿它當安全 runtime。實作是 Rust（約 89%、19 個 crate 的 workspace）。要拿它上 production 多租戶場景，現在還太早。

## 2. 架構三件套：Gateway、Supervisor、CLI

```
User 介面 (CLI / SDK / TUI)
        │ gRPC / HTTP (mTLS / OIDC)
        ▼
   Gateway 控制平面 ── 持久化 / 憑證 / 身分
        │ 供裝 workload
        ▼
   Sandbox 資料平面
   Supervisor (root) ──spawn＋限制──► Agent child (非 root)
        │ 所有 egress
        ▼ Policy proxy ── allowed ─► 外部服務
                       └ inference.local ─► 模型後端
```

- **Gateway（控制平面）**：認證後的單一入口，管 sandbox 生命週期與認證邊界，持久化所有物件（protobuf object store、SQLite 預設、Postgres 上 HA）。注意一個刻意的設計：**Gateway 不做 per-request 網路決策**——那發生在沙箱內，因為只有沙箱看得到本地的 process identity。
- **Supervisor（沙箱內的安全邊界，真正的執法者）**：以 root 啟動，先把隔離全準備好、跑起 policy proxy、注入憑證、開好回連 socket，**降權之後**才啟動 agent child（非特權 user）。還有一個漂亮的**反向連線設計**：supervisor 永遠 outbound 連回 Gateway，Gateway 不需要 dial 沙箱的 IP，connect / exec / 檔案同步全走這條 relay——省掉 NAT、port mapping、bridge 的麻煩。
- **CLI / SDK / TUI**：`openshell` CLI 是主入口，`openshell term` 是 k9s 風格的即時儀表板。

## 3. 真正的功夫：實際疊了六層的 kernel 沙箱

官方文件講「三層核心機制」（seccomp、Landlock、network namespace），但讀原始碼會發現實際疊了**六層**。核心模式是這樣：**supervisor 維持 root 把所有限制「準備好」，fork 出 agent child 後，在 `pre_exec`（fork 之後、exec 之前那個瞬間）一次套上所有限制並降權**，agent 之後永遠回不到 root。

| 層 | Kernel primitive | 怎麼做 |
|---|---|---|
| **Filesystem** | Landlock | 兩階段：root 階段先開好 PathFd，降權後 `restrict_self()` 上鎖（不可逆、不理 capability） |
| **Syscall** | seccomp-BPF | default-allow + 針對性封鎖，先 `prctl(PR_SET_NO_NEW_PRIVS)` |
| **Network 隔離** | network namespace | `ip netns` 建 `sandbox-{uuid}` + veth pair，child 在 pre_exec `setns(CLONE_NEWNET)` 進去 |
| **Egress 強制** | nftables | output hook 只放行「到 proxy」「lo」「established」，其餘 v4+v6 全 reject——**逼所有流量走 CONNECT proxy 的關鍵** |
| **Mount 隔離** | mount namespace | `unshare(CLONE_NEWNS)` 把憑證 socket 藏起來 |
| **降權** | setuid/setgid | pre_exec 裡切到 sandbox user 後，**驗證 `setuid(0)` 必須失敗**——成功代表沒降乾淨，直接 abort |

最後那個 `setuid(0)` 必須失敗的自我檢查很值得一提：它不是假設降權成功，而是主動試著變回 root、確認失敗才繼續。這種「不信任自己」的寫法正是 sandbox 該有的偏執。

**seccomp 封鎖清單**也看得出用心。它擋掉 raw / kernel socket（防 agent 開原始 socket 繞過 proxy）；`AF_NETLINK` 只放行 `NETLINK_ROUTE`（給 `getifaddrs()` 用，否則 Node / Python / Go 一啟動就掛），其餘 netlink 全 `EPERM`。無條件擋掉的有：mount 家族、`bpf`、`perf_event_open`、`userfaultfd`、模組載入、`kexec`、`ptrace`、`io_uring`。還有幾個條件式的細節很巧：`clone3` 直接用 `ENOSYS` 擋掉，逼 glibc fallback 回舊的 `clone`——因為 `clone3` 的 flag 藏在指標結構裡，BPF 讀不到、無法在 syscall 入口判斷，乾脆讓它「看起來不存在」。另外有個 **bypass monitor** 讀 `/dev/kmsg` 解析 nftables 的 LOG，把 agent 嘗試直連繞過 proxy 的行為記成結構化事件。

## 4. Landlock 深拆：為什麼檔案那層非它不可

六層裡的檔案那層用的是 **Landlock**，這是整套設計的地基，值得單獨拆。

### Landlock 是什麼

一句話定位：**Landlock 是讓非特權程序自我設限的 Linux 核心機制**——程序宣告一組規則（哪些路徑唯讀、哪些可讀寫，其餘預設拒絕），套上之後連自己都解不開。運作分兩階段：`landlock_create_ruleset` 開一張空 ruleset → `landlock_add_rule` 逐條加路徑權限 → `landlock_restrict_self` 套到當前 thread，**之後就回不去了**。沒有「解除沙箱」的 API，這是刻意的。

三個關鍵性質：

1. **免 root**。chroot 要 root、AppArmor policy 要管理員預寫；Landlock 不用——任何一般使用者程序自己呼叫上面三個 syscall 就能把自己關起來。安全收斂變成應用程式自帶的能力。
2. **不可逆，且不理 capability**。`restrict_self()` 之後規則只能愈來愈嚴，**即使後續拿到 `CAP_SYS_ADMIN` 也逃不出去**。
3. **以核心物件為單位**。是 allowlist，不是 blocklist。

### 為什麼是 Landlock 而不是 mount namespace

這是 OpenShell 一個很容易被滑過、但其實很關鍵的決定。直覺上，把 agent chroot / pivot_root 進一個小目錄不就好了？問題出在前提：**沙箱容器本身就帶著 `cap_add SYS_ADMIN`**（supervisor 建 netns、mount ns 都要用）。一旦你用 mount namespace 當檔案牢籠，被劫持的 agent 用同一個 `SYS_ADMIN` 就能再 `mount` / `pivot_root` 把自己搬出去——牢籠的鑰匙等於掛在犯人手上。

兩者本質不同：

| 面向 | mount namespace | Landlock |
|---|---|---|
| 本質 | 改「看得到哪些 mount」（可見性） | 對既有 fs 做「能不能存取這條 path」（存取控制） |
| 權限 | pivot_root/mount 需 `CAP_SYS_ADMIN`，得趁 root 做 | `restrict_self()` 免權限，降權後 exec 前套上 |
| 唯讀粒度 | `mount -o ro,bind`，粗、per-mount | 分 READ/WRITE，per-path |
| 逃逸抗性 | 殘留 `CAP_SYS_ADMIN` / 外洩 fd 可繞 | 不可逆、跨 exec/setuid，專為關非特權程序設計 |

`restrict_self` 不可逆、不理 capability，有沒有 `SYS_ADMIN` 都繞不過——這正是它在「容器本就帶高 cap」的前提下唯一說得通的選擇。代價是 Landlock 硬綁現代 kernel（5.13+），這個代價第 7 節會回來算。

### ABI 版本：Landlock 早就不只管檔案

`go-landlock` 程式碼裡常見的 `landlock.V5`，那個 `V5` 不是套件版本號，是 **Landlock 的 ABI 版本**。Landlock 自 kernel 5.13 起以 ABI 逐步長能力，每一版對應夠新的 kernel：

| ABI | Kernel | 加了什麼 |
|---|---|---|
| v1 | 5.13 | 檔案系統存取控制（讀／寫／執行／刪除，路徑級） |
| v2 | 5.19 | `LANDLOCK_ACCESS_FS_REFER`——跨目錄 rename / link |
| v3 | 6.2 | `truncate` 操作 |
| v4 | 6.7 | **TCP `bind` / `connect` 限制**（指定允許的 port）——開始管網路 |
| v5 | 6.10 | 對字元／區塊裝置的 `ioctl` 限制 |

重點：**v4 起 Landlock 已不只是「檔案沙箱」**，能限制程序只准 connect 哪些 TCP port。所以要 `V5` 的完整能力就得 kernel 6.10。

### 跟 seccomp 是互補，不是二選一

**seccomp 過濾 syscall 編號與參數**（「禁 `mount`」），看的是「你呼叫了哪個 syscall」；**Landlock 看的是「你想碰哪個路徑、哪個 port」**，是 path-aware 的。seccomp 表達不了「能 `open`，但只能 open 這個目錄」，Landlock 正好補上。所以 OpenShell 兩個疊著用——這就是上一節六層裡 Filesystem 和 Syscall 各佔一格的原因。

## 5. 網路、憑證、會自我演進的 policy

如果說檔案那層是地基，OpenShell 真正的招牌在網路與憑證。

**Default-deny egress + L7 policy**：每個沙箱開出來幾乎沒有對外網路，要開得寫 policy。而且 proxy 不只做 L4 的 host:port allowlist——它**終止 TLS**（用沙箱的 ephemeral CA）做 L7 檢查：能精確到「允許 `GET api.github.com`，但擋掉 `POST`」。規則還能綁 **binary identity**（只有 `/usr/bin/curl` 能連、別的 binary 不行），並硬擋內網 IP、link-local、cloud metadata（`169.254.169.254`）防 SSRF。無規則 = deny，explicit deny 永遠贏過 allow。

**Inference router / Privacy router（最有特色）**：`https://inference.local` 是個特殊端點，proxy 用沙箱 CA 終止本地 TLS、辨識 OpenAI / Anthropic 格式、**剝掉 caller 自帶的憑證**，再用 Gateway 給的 route bundle 轉發到真後端。效果很反直覺也很漂亮：**agent 程式碼裡根本不需要、也拿不到模型 API key**——憑證留在 Gateway，沙箱只收到一個 minted token。就算 agent 被 prompt injection 拐了，它也偷不到本來就不在它手上的金鑰。支援 OpenAI / Anthropic / NVIDIA / Google Vertex。

**Providers（憑證束）**：所有憑證以具名 provider 管理，建沙箱時注入成環境變數，**從不落地到沙箱檔案系統**。進階還能走 SPIFFE Workload API 換 OAuth2 的 dynamic token grant。

**Policy Advisor + Z3 Prover（會自我演進的 policy）**：這是小編覺得最聰明的一塊。agent 撞到 deny 不會卡死，而是進入一條提案迴路——proxy 把 denial 事件回報 Gateway，自動產生一條**最窄**的 policy 提案。但提案不會直接生效：有一個用 **Z3（formal solver）做形式化驗證的 prover**，會檢查這條新規則是否擴大攻擊面（是否新增 link-local 可達、是否讓沒憑證的 binary 碰到有憑證的 host、是否新增 HTTP method）。**任何一個 finding 就擋掉自動核准**，預設一切落 pending 等人審。用定理證明器來守 sandbox policy 的攻擊面，這個工程品味相當高。

安全相關行為都走 **OCSF**（Open Cybersecurity Schema Framework）結構化事件輸出，而且**永不記 secret / token / query param**。

## 6. 容器還是 MicroVM：兩種隔離強度

上面整套（namespace + Landlock + seccomp + nftables + 降權）在容器 driver（Docker / Podman / Kubernetes）下是**疊在容器裡**的。但 OpenShell 還有一個 **VM driver**：用 **libkrun** 走 KVM 硬體虛擬化（macOS 上走 HVF），開出真正的 microVM。這是給「連容器邊界都不信任」的場景用的硬隔離。同一套 policy 語意，driver 只是把它翻譯成各平台的原生操作。GPU 還能透過 VFIO passthrough 餵進沙箱，給地端模型推論用。

這裡有一個很反直覺的部署前提值得知道：**要在容器內建出強沙箱，那個容器本身反而需要較高權限**——Docker driver 要 `cap_add: SYS_ADMIN, NET_ADMIN` 加 `apparmor=unconfined`（否則 docker 預設的 AppArmor 會擋掉 supervisor 要做的 mount）。所以在嚴格的 K8s Pod Security Standards（restricted）叢集裡，這些會被 admission 擋下，要嘛開例外、要嘛改用 MicroVM driver。你不是把 agent 丟進一個被綁死的容器，而是**給容器較大的 caps、信任 supervisor 會把 agent 重新關好**。

## 7. 那塊地基不一定在：BestEffort 靜默降級

第 4 節說 Landlock 的代價是「硬綁 kernel 5.13+」，現在來算這筆帳——這也是部署 OpenShell（或任何依賴 Landlock 的東西）最容易踩的坑。

Landlock 不是「kernel 夠新就一定有」。它是**核心編譯選項加開機時的 LSM 設定**（`CONFIG_SECURITY_LANDLOCK` 加上 `lsm=` 開機參數要把它排進去）。所以你完全可能跑在一個 kernel 5.13+、看起來夠新的系統上，但它的 LSM stack 裡**根本沒有 landlock**——一些嵌入式、NAS 級或廠商客製的 Linux build 就是這樣，stack 只有 `capability,apparmor`，Landlock 沒被編進去。而且容器共享 host kernel，所以 Landlock 的強度直接看 host kernel，沙箱自己無能為力。

這時候 go-landlock 那個 `BestEffort()` 的「靜默降級」就從貼心變成陷阱：在缺 Landlock 的環境，它會**自動 drop 套不上的規則繼續跑、而不是報錯**，你的檔案隔離那層於是**悄悄退回只剩 DAC**（Discretionary Access Control，傳統 Unix rwx），程式照跑、零 error，但那道檔案牆根本沒立起來。一個 world-writable 或 agent 自己擁有的路徑，DAC 不會幫你擋。

OpenShell 對這點處理得不錯：它在啟動時如果偵測到 Landlock 不可用，會印出一條 `FINDING: Landlock Unavailable [HIGH]`——讓你知道你少了哪一層，而不是默默假裝它在。對應的反面教訓很直接：**`BestEffort()` 之後一定要驗證實際生效的強度，別假設它套上了**；要嘛接受降級、要嘛改用 `hard_requirement` 讓建沙箱直接失敗、要嘛走 microVM 那條不靠 host LSM 的路。

## 8. 旁證：連 Go 標準庫都要把 Landlock 收進去

回到開頭那個 Go 提案 #68595。它的動機場景跟 OpenShell 幾乎是同一句話：一個 Go 程式對**不可信的輸入**反覆呼叫外部 `git`，數十萬次，每次只需要碰一個暫存目錄，卻繼承了父程序的全部檔案存取權。提案的做法是擴充 `syscall.SysProcAttr`，讓 `os/exec` 直接吐出受 Landlock 限制的子程序：

```go
cmd.SysProcAttr = &syscall.SysProcAttr{
    UseLandlock:     true,
    LandlockRuleset: fd, // 用 github.com/landlock-lsm/go-landlock 建
}
```

三個設計重點呼應前面所有討論：**只限制子程序**（父程序當 orchestrator、子程序當 untrusted worker，正是 OpenShell 的 supervisor/agent 形狀）、**向後相容**（沒設就沒事）、`no_new_privs` 與 Landlock **正交**。值得一提的是這提案是更早 #47049（已撤回）的續命版——標準庫為它試了兩次。OpenShell 在 runtime 層押 Landlock、Go 在語言層也要把它做進標準庫，兩個獨立方向同時指向同一個機制，這個訊號比任何單一專案的背書都強。

## 9. 小編的看法

小編認為真正的故事是：**Landlock 正從 security 圈的 niche 機制，變成基礎建設的預設選項**，而 OpenShell 示範了把它（以及 seccomp、nftables、namespace）組裝成一套完整 agent 沙箱該長什麼樣。

幾個值得帶走的判斷：

- **least privilege 不該把賭注全押在單一層**。OpenShell 疊六層不是為了好看——是為了某一層失效時其他層還在。第 7 節正好示範：平台沒有 Landlock 時檔案隔離蒸發，這時撐住場面的是 seccomp + nftables + 降權。務實的問法不是「有沒有沙箱」，而是「**每一層各擋掉什麼、哪一層在這個平台上其實是空的**」。
- **把憑證從 agent 手上拿走，可能比任何過濾都有效**。inference router 那招的精神是：與其偵測 agent 會不會洩漏 key，不如讓 key 一開始就不在它手上。這對防 prompt injection 的 blast radius 很有啟發——**OpenShell 限制的是「被騙的 agent 能造成多大破壞」，它不偵測 injection 本身**，這兩件事要分開看。
- **基礎設施層 vs application 層的分離是對的方向**。policy 放在 agent 構不到的地方、用 formal prover 守住每次放寬，這比「一開始就把權限開好」安全得多。

最後拉高一點：這整件事是 defense-in-depth 老原則的具體演出——把最小權限從運維手動配置，往下推到 runtime 自帶（OpenShell）、再推到語言標準庫（Go）。方向小編很看好。但有個現實要記著：OpenShell 自己還是 alpha、多租戶未到位；而它最仰賴的 Landlock 那層，偏偏在舊 kernel / 客製平台上會「靜默降級」。所以真要拿它擋事，部署前務必實測每一層到底有沒有生效——別讓一句「secure by default」騙了自己。

### 參考連結

- [NVIDIA/OpenShell · GitHub](https://github.com/NVIDIA/OpenShell)（Apache-2.0）｜ [官方 docs overview](https://docs.nvidia.com/openshell/about/overview)｜ [NVIDIA Blog：Secure Autonomous AI Agents With OpenShell](https://blogs.nvidia.com/blog/secure-autonomous-ai-agents-openshell/)
- [golang/go#68595 · syscall: support process sandboxing using Landlock on Linux](https://github.com/golang/go/issues/68595)（前身 [#47049](https://github.com/golang/go/issues/47049)，已撤回）
- [github.com/landlock-lsm/go-landlock](https://github.com/landlock-lsm/go-landlock) — 建立 ruleset 的 Go 套件
- [Landlock 官方文件](https://docs.kernel.org/userspace-api/landlock.html) — kernel.org 的 unprivileged sandboxing 說明
