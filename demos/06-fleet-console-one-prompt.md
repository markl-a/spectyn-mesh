# Demo 06 — One prompt, every AI CLI on every device（一個 prompt，全艦 AI CLI 協同）

**長度（Length）**：85 秒（實錄，非腳本）
**狀態（Status）**：🟢 已錄製 2026-09-14，真機、無剪輯（frames captured from the running Windows App）
**錄影**：[`mesh-console-one-prompt-2026-09-14.gif`](mesh-console-one-prompt-2026-09-14.gif)

<p align="center">
  <img src="mesh-console-one-prompt-2026-09-14.gif" alt="Spectyn Mesh 主控台 — one prompt fanned out to 12 AI CLIs / app-local executors on 8 devices, every tile returns its own sub-task, output and signed receipt" width="900">
</p>

## 鉤子（Hook）

> "Type one prompt on any device. The coordinator splits it into one sub-task per AI CLI across the whole fleet — Windows, macOS, iOS, Android — and every tile comes back with its own answer and a verified receipt."
>
> 在任一台裝置打一個 prompt，coordinator 拆成每個 AI CLI 自己那一份，Windows／macOS／iOS／Android 一起跑，每格帶回自己的答案與驗證過的收據。

## 畫面上有什麼（Cast / setup）

The recording is the desktop **主控台（fleet console）** of the Windows App on the coordinator machine. Every device the account owns is a card; every AI CLI or app-local adapter on it is a row.

| Device | Platform | Adapters in this run |
|---|---|---|
| z13 (coordinator) | Windows | claude, agy |
| ayaneo | Windows | claude, agy |
| acer laptop | Windows | claude |
| m1 | macOS | claude, agy |
| m5 | macOS | claude, agy |
| iphone | iOS | app-local (Groq, BYOK) |
| ipad-pro | iPadOS | app-local (Groq, BYOK) |
| android tablet | Android | app-local (Groq, BYOK) |

12 participants on 8 devices. `codex`, `opencode` and `copilot` were deliberately unticked at freeze time and appear as **exclusions** in the sealed manifest (`partial-fleet-demo`), which is the honest way the contract records "present but not used".

## 發生了什麼（What the 85 s show）

| 時間 | 畫面 |
|---|---|
| 0:00 | Roster frozen: sealed manifest (sha shown), 12 adapters, 13 exclusions listed by name and reason. |
| 0:03 | Prompt typed at the bottom: *幫我設計一個用 Python 寫的每日天氣通知小工具：抓取資料、排程、通知方式、錯誤處理、測試。每人只做自己那一份，三句以內。* |
| 0:05 | Confirmation dialog: the exact prompt, `prompt_sha256`, the 12 participants, `participants_sha256`, manifest sha — what is sent is what you see. |
| 0:08 | `送出 fanout`. The coordinator's planner (`groq-free`) splits the prompt into **12 distinct sub-tasks** — data fetch, scheduling, notification, error handling, logging, tests, CI… one per adapter. |
| 0:10–0:45 | Tiles flip 閒置 → 執行中 → 完成 as each CLI / phone finishes. Each tile shows **its own sub-task**, **its own output**, the challenge nonce it had to echo, and `receipt ✓ completed`. |
| 0:45–1:25 | Scroll through all eight device cards: Windows CLIs, macOS CLIs, and the iPhone / iPad / Android app-local rows that ran inference through the phone's own Groq key. |

Coordinator-side result for this exact task (`f9545be7`): **state `completed`, 12 / 12 attempts completed with verified receipts, 12 distinct sub-tasks, planner `groq-free`.**

## 為什麼這是旗艦（Why it matters）

- **One input, whole fleet.** The prompt was typed once. No SSH, no per-machine terminals, no copy-paste between devices.
- **Real decomposition, not broadcast.** The planner gave every adapter a different job; the tiles prove it.
- **Phones are executors, not just remotes.** iPhone, iPad and Android each pulled their sub-task and ran it with a device-local API key (BYOK); their receipts are `app-local`, signed and verified like the desktop ones.
- **Honest by construction.** Exclusions are named with a reason; the manifest and the fanout body are hashed and shown before sending; a task only reads 全部完成 when every attempt completed *and* its receipt verified.

## 錄製方式（How it was captured）

Frames were captured every 0.7 s from the live App window (Windows `PrintWindow`), the left navigation and OS chrome were cropped, unchanged frames were dropped, and the result was written as a GIF. No cuts, no re-ordering. Device names and tailnet addresses are real; the cluster secret never appears on screen.
