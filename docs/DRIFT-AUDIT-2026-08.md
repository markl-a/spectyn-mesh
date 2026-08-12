# DRIFT-AUDIT — 文件宣稱 vs 實作落差（2026-08）

> 針對 `docs/ARCHITECTURE.zh-TW.md`（v0.6.0，🟢/🟡/🧪 誠實標記）所做的實作對照稽核。
> 方法：7 個獨立 reviewer 各讀一個子系統的 `core/src/` 原始碼，逐條驗證文件宣稱，
> 記錄 PRESENT / PARTIAL / MISSING / STUB 與 file:line 證據、測試存在與否。
> 稽核範圍為**公開鏡像 `spectyn-mesh`**（落後私有主 repo）；行號對應稽核當日 HEAD，之後可能位移。

## 總結

**這份架構文件屬誠實系**——核心機制大多真的存在、有測試，🟢/🟡/🧪 標記方向大致正確。
落差幾乎都是三類，**而非憑空捏造**：

1. **零件真、整合未接** — 子系統程式碼真且有測試，但未 wire 進 production live 路徑，或預設關閉。
2. **措辭比程式碼強** — signed→實為 MAC、synthesizes→實為 collect、5xx→實為 429/503、auto→不存在、3 段 gate→實為 2 段。
3. **文件彼此打架 + 陳舊註解** — 尤其加密；多處 `unimplemented!() Stage 4 stub` 註解下面其實已是真實作。

## 落差主表

| # | 子系統 | 標記 | 實測 | 關鍵落差 |
|---|--------|:--:|:--:|--------|
| 1 | Agent 核心迴圈 | 🟢 | 大致名副其實 | `stall 偵測`只 log、無控制流效果、零測試；compaction 歸錯檔（`context.rs` 是死碼） |
| 2 | Providers/resolver | 🟢 | 機制真、敘述多處錯 | 失效轉移/熔斷/retry 不在 `resolver.rs`；「5xx」實為 429/503；額外 provider 的 flag 鎖是漏的 |
| 3 | Tool gate + 權限 | 🟢 | 核心真、非「3 段」 | 每次呼叫只過 2 段；trust 預設 enforcement=Off；capabilities 是獨立叢集路由 |
| 4 | 治理 + 飛行紀錄器 | 🟢 | 大致名副其實 | 「signed」實為 HMAC hash-chain（對稱 MAC，非數位簽章）；「auto」模式不存在；contract gate 預設 OFF |
| 5 | 身分/加密 | 🟢 | 加密真、keystore 弱 | OS keystore 未接到真正在用的 root identity.key；三份文件自相矛盾；兩套 HKDF 衍生規制 |
| 6 | Mesh/swarm/fleet | 🟡 | 🟡 對、§6 敘述灌水 | swarm「synthesize」實為收集；routing「load」只是 active-task 計數；fleet 單機一次一 spec、非多機連續迴圈 |
| 7 | 遠控 + MCP | 🧪/🟢 | 標記校準準 | MCP 名副其實；遠控是「零件齊、整合未完」（Telegram 實接 EchoDispatcher、Slack 未 mount、WhatsApp 樁） |

---

## 逐子系統證據

### 1. Agent 核心迴圈（§2，🟢）

| 宣稱 | 結果 | 證據 |
|------|:--:|------|
| Turn cap | PRESENT | `agent.rs:852` `for round in 0..max_rounds`；default 25（`config.rs:80`）；env `SPECTYN_MAX_ROUNDS` |
| Stall 偵測 | **PARTIAL（誇大）** | `STALL_THRESHOLD=2`（`agent.rs:75`）但只 `tracing::warn!`（`agent.rs:940`），**不改控制流**、**零測試** |
| Context compaction | **PRESENT 但歸錯檔** | 真正跑的是 `agent.rs:3007 compact_if_needed`；文件說的 `context.rs:62 compact_conversation` 是**死碼**、無 caller、無測試 |
| JSONL session 持久化 | PRESENT | `session.rs` `ConversationStore`，`~/.spectyn-mesh/conversations/<id>.jsonl`；測試齊 |
| `/compact` LLM 摘要 | PRESENT | `session.rs:594 compact_via_llm` → 原子 rewrite；接 TUI/REPL |
| Workspace scope | PRESENT | `context.rs:652 WorkspaceContext`；測試齊 |
| 每輪+累計成本 | PRESENT | `cost.rs CostTracker`，落盤 `costs.json`；測試齊 |

**評估**：🟢 大致站得住。兩個誇大：stall 偵測是 log-only 診斷、compaction 被歸給死碼的 `context.rs`。

### 2. Providers / resolver（§3，🟢）

| 宣稱 | 結果 | 證據 |
|------|:--:|------|
| 單一 trait 抽象 | PRESENT | `LlmProvider` trait；`resolver.rs::build_provider`（167-221）純 dispatch |
| 額外 provider 鎖在 flag 後 | **PARTIAL（漏洞）** | 8 模組確有 `#[cfg(feature="experimental-extra-providers")]`，但 `resolver.rs:811-826` `OpenAICompatProvider::url()` 有**無 cfg** 的 mistral/xai/together/… 端點；免 flag 即可發真流量 |
| 額外 provider 為完整 adapter | **PARTIAL** | 8 個只有 `streaming_url/auth_header/health_check`，**無 `complete()/stream()`**；`cohere.rs:11` 自述「V1 只出 metadata + health-check ping」 |
| 失效轉移順序在 `resolver.rs` | **MISSING（歸錯檔）** | resolver 零 failover 邏輯；真正在 `providers_wire.rs`（`load_fallback_chain`）+ `agent.rs:1998+` |
| 429/5xx retry+backoff | **PARTIAL** | `retry.rs` 真且有測試，但 `is_retryable_status`（286）只 `429\|503`；500/502/504 不 retry |
| 熔斷器 | PRESENT | `circuit_breaker.rs` 真 Closed→Open→HalfOpen 狀態機；接 `providers_wire.rs:1416` |
| credential_scanner 偵測登入 CLI | **PARTIAL** | `scan_all()` 只偵測 3 個 env key（OPENAI/ANTHROPIC/OPENROUTER），**不掃 CLI 登入** |
| CLI 後端不存 API key | PRESENT | `cli_session_provider.rs:421` 忽略 `_api_key`；PTY 橋驅動已登入 CLI |

**評估**：機制真、🟢 大致可辯護，但 §3 段落有四處具體 drift（歸錯檔、5xx 誇大、flag 鎖漏、scanner 誇大）。

### 3. Tool gate + 權限（§4，🟢）

| 宣稱 | 結果 | 證據 |
|------|:--:|------|
| 單一 process-level gate，每次呼叫都過 | **大致 PRESENT** | choke point `tools/mod.rs:246 execute()`→`gate_check()`；每個 entrypoint 都 install（`main.rs:24`、`bin/spectyn.rs:3882`）；MCP 與 skill bash 兩繞道已用 `gate_allows()` 補上（`mcp.rs:174`、`skill_executor.rs:245`） |
| permission.rs Tool(specifier) allow/ask/deny | PRESENT | 859 行、**26 測試**；deny>ask>allow；wildcard/glob/domain 比對；redirect/chain 降級 |
| project_trust 逐目錄 | **PRESENT 但預設 OFF** | 真且 14 測試，但自述「Phase 2b skeleton」，`TrustPolicy::default()==Off`（純 pass-through） |
| capabilities required_caps + 節點宣告 + 拒絕 | **PRESENT 但非 tool gate** | 真，但屬 `cluster_dispatch_wire.rs` 的**叢集任務路由**；per-call gate 只有 2 段 |
| ~60 工具 | PRESENT | 實測 61（`tools/mod.rs:263-374`） |

**評估**：核心真且測試多，但文件的「單一 3 段式 gate」不成立——每呼叫只 2 段、trust 預設空轉、capabilities 是別的子系統。

### 4. 治理 + 飛行紀錄器（§7，🟢）

| 宣稱 | 結果 | 證據 |
|------|:--:|------|
| 風險分級（execute_high 等） | PRESENT | `execution_contract.rs:33-77 RiskLevel`；`decision.rs classify_event`；有測試 |
| 審批佇列 approve/stop | **PARTIAL（廣度）** | `pending_approvals.rs` + phone/OS/Telegram/inbox 真；文件說的 **TUI/Web console 審批 UI 在此範圍找不到** |
| 「signed」逐字稿 | **PARTIAL（措辭誇大）** | 實為 **HMAC-SHA256 hash-chain**（`recorder.rs:235`，HKDF 衍生對稱金鑰）；防竄改真、但**非數位簽章**，持裝置金鑰者可偽造、無公開驗證 |
| enforcement auto vs pre_action_blocking | **PARTIAL（命名 drift）** | 真枚舉為 `PreActionBlocking/PreActionDelegated/PostActionObserved`；**無 `auto` 值**；pre_action_blocking 完整且測試 |
| deny-until-approved | **PRESENT，但 live gate 預設 OFF** | 狀態機真（`execution_contract.rs:245`）；`contract_gate.rs` 需 `SPECTYN_CONTRACT_GATE=1`，預設 pass-through |

**評估**：🟢 大致站得住。兩個誇大：「signed」（實為 MAC）、「auto」（不存在）；TUI/Web 審批面與 live contract gate 弱於敘述。

### 5. 身分 / 加密（§8，🟢）

| 宣稱 | 結果 | 證據 |
|------|:--:|------|
| 64-byte root IKM | PRESENT | `identity.rs:59` `OsRng` 填 64 bytes、O_EXCL 0600、zeroize；測試齊 |
| 獨立 ed25519 簽章金鑰 | PRESENT | `keys/ed25519.priv` 獨立於 IKM；`ed25519-dalek`；測試齊 |
| age 加密事件內容 | **PRESENT 且已 wire** | owned-memory 恆開（`life_node/storage.rs:124`）；SPEC-16 wire（`capture_*_wire.rs`）。**caveat**：`write_event` 收「已加密 bytes」，加密正確性靠各 caller（已審核之 production caller 皆先加密） |
| HKDF-SHA256 衍生 | **PRESENT 但兩套規制** | `life_node/key_derivation.rs`（label `.event-encryption-v1`，IKM=全 64B）與 `encryption_wire.rs:248`（label `.v1.event-encrypt`，IKM=前 32B）**產不同金鑰**；legacy 路徑已停用 stub |
| OS keystore 已 landed | **MISSING（最大落差）** | mac/win/android backend 程式碼**都真**，但**未接到實際在用的 root identity.key**（mac 仍 0600 檔；只有 **Windows DPAPI** 真 at-rest 包）。三方矛盾：ARCHITECTURE 說「landed🟢」、`ENCRYPTION-STATUS.md` 說「v0.7.0 未完」、程式碼註解說「unimplemented Stage 4」 |
| byte-identical 還原 | **PARTIAL** | identity.key 檔 round-trip 有測試（`cuj05_identity_import.rs`）；但**無單一端到端測試**證「還原後解出同一 plaintext」，屬傳遞式證明 |
| broker vault wrap | PRESENT | `broker_vault_wire.rs` age v1 + HMAC + JWT + X25519 wrap 全真（檔頭 stub 註解為 stale） |

**評估**：加密本體真且已 wire，🟢 可辯護；**keystore 是弱點**（程式碼在、但未接 root key、且三方文件打架）。

### 6. Mesh / swarm / fleet（§6，🟡）

| 宣稱 | 結果 | 證據 |
|------|:--:|------|
| HMAC cluster secret + fail-closed | PRESENT | `mesh.rs:1948` 真 HMAC + 常數時間比對；`auth_gate.rs:90` 未設→403。escape hatch `SPECTYN_ALLOW_EMPTY_CLUSTER_SECRET=1`（預設關） |
| peer health + load/capability routing | **PRESENT 但 naive** | `mesh.rs:278 select_peer`（80 測試）；「load」= active-task 計數 + 失敗連續數，非真資源負載 |
| swarm 扇出 + **綜整** | **扇出 PRESENT，綜整 MISSING** | 回傳 `Vec<PeerOutput>` 原始輸出；**無任何 merge/synthesis** |
| crew 多 CLI 管線 | PRESENT | `bin/spectyn.rs:4353`；codex 實作/claude 審/agy 稽核，2 approval 落地；~58 測試 |
| fleet 原子認領→實作→驗證→交叉審查 | **REAL 但單機/一次性/驗證部分為樁** | 原子認領（`fleet/queue.rs:150` CAS）+ 交叉審查（`fleet/gate.rs`）真且測試多；但 **單機**（自述 single-machine）、**無連續迴圈**（僅 `run --once`）、預設 `L1Executor` 的 build-verify 是樁（只有 `--executor crew` 真跑 `cargo build`） |

**評估**：projects-root 的「fleet 劇場/refs=0」**在程式碼層站不住**，🟡 比「劇場」貼近真相。但 §6 敘述**規模灌水**：應改為 single-node / one-shot / diff-review-only，並拿掉「synthesizes」與多機自主暗示。

### 7. 遠控 + MCP（§11 🧪 / §10 🟢）

| 宣稱 | 結果 | 證據 |
|------|:--:|------|
| Telegram LIVE（long-poll/dispatcher/media） | **PARTIAL** | long-poll 真（`telegram.rs:170`）；但 live launcher 接 `EchoDispatcher` 非 agent（`spectyn.rs:5015`，註解「暫代到 PR #52」）；media 程式碼真但 live loop 只讀 text |
| Slack LIVE（postMessage + HMAC inbound） | **PRESENT 但未接 serve** | `SlackBot::send_message`、HMAC webhook（過官方測試向量）皆真；但**無任何 serve/CLI 掛載**（grep 0 命中） |
| WhatsApp 樁 | STUB（誠實） | `whatsapp.rs:108` 恆回 `NotImplemented`，自述 STUB |
| Channel trait + persona + token-bucket | **PARTIAL** | trait/persona 真；rate limiter 真但**只被兩個恆失敗的 stub 呼叫**，真流量（SlackBot/Telegram）不過限流器 |
| MCP server stdio + /mcp | PRESENT | `mcp.rs:34 run_stdio` + `serve.rs:232 /mcp`（HMAC-gated）；接 production |
| MCP client 外部 server 變工具 | PRESENT | `mcp_client.rs` 真 handshake + `<server>_<tool>` 前綴 + keepalive |

**評估**：🧪/🟢 標記**校準準**。MCP(🟢)名副其實；遠控(🧪)是「零件齊、整合未完」——信標記勝過信 prose 的讀者會得到正確預期。

---

## 建議修正優先序

| 優先 | 項目 | 動作 |
|:--:|------|------|
| 🔴 高 | 加密文件三方矛盾 + keystore 未接 root key | 統一 ARCHITECTURE §8 / ENCRYPTION-STATUS / 程式碼註解；明講 mac root key 仍 0600、只有 Win DPAPI 真 at-rest 包 |
| 🔴 高 | 「signed transcript」正名 | 改為 HMAC-SHA256 hash-chain / tamper-evident（非數位簽章、無不可否認性） |
| 🟡 中 | extra-provider flag 漏洞 | 把 `OpenAICompatProvider::url()` 的額外 provider 端點也 cfg-gate，或修正「鎖在 flag 後」敘述 |
| 🟡 中 | 死碼 + stale 註解 | 移除/標註 `context.rs::compact_conversation`；清掉多處 `unimplemented!() Stage 4 stub` 陳舊註解 |
| 🟡 中 | §6 敘述降級 | swarm 拿掉「synthesizes」、routing「load」講清是 active-task 計數、fleet 標 single-node/one-shot、預設 executor build-verify 是樁 |
| 🟢 低 | §4「3 段 gate」正名 | 每呼叫 2 段（permission + trust）；capabilities 另述為叢集路由；trust 預設 Off |
| 🟢 低 | stall 偵測 | 標為 log-only 診斷，或補控制流 + 測試 |

---

*稽核方法：7 平行 reviewer，各讀單一子系統原始碼對照 `docs/ARCHITECTURE.zh-TW.md`，記錄 file:line 證據與測試存在與否。本報告只描述落差，不含修正實作。*
