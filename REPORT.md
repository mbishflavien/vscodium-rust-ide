# Static Audit â€” `mbishflavien/vscodium-rust-ide-amd`

- **Commit:** `afd1c686412945bd2401af1375dc492b8be9bc94` (`feat(kortex): GAC + KDKVC + CCET inference stack`)
- **Fork of:** `H4D3ZS/vscodium-rust-ide-amd` (identical tree; fork adds no commits)
- **Method:** static only. Nothing was installed, compiled, or launched.
- **Date:** 2026-11-03

---

## 1. Executive summary

This is a **single-developer Tauri 2 + React 19 desktop application that imitates VS Code**, not a "VS Code rewritten from the ground up in Rust" as the README claims. The editor surface is **Monaco** (the same editor component VS Code ships), wrapped in a custom React shell, with a vendored copy of VS Code's own web assets in `src/public/vs/`. The Rust side is a ~29 kLOC backend that is genuinely broad â€” 285 registered Tauri commands â€” but sprawling and largely unverified.

The one part with real engineering substance is the **Kortex inference stack** (`src-tauri/src/kortex_gac`, `kortex_kvcache`, `src/kortex/ccet.ts`): it actually parses GGUF via `candle-core`, computes per-tensor geometry, plans CPU/GPU tiering, spawns `llama-server`, and implements a SHA-256 disk KV-cache proxy over the llama.cpp slot API. Its unit tests are real. Its performance claims are not reproduced anywhere.

Everything else falls into three buckets: (a) real but ordinary IDE plumbing (PTY terminals, LSP, Git, MCP, Monaco), (b) **fabricated capabilities that return hardcoded fake data** (nmap scans, exploit lookup, "0-day generation"), and (c) **features that are actively hostile to the user's privacy** â€” a default third-party inference endpoint, a hardcoded bearer token, cookie capture from `claude.ai`/`gemini.google.com`, a MITM proxy, and a credential harvester that tests leaked keys against live provider APIs.

**It does not build.** `src-tauri/Cargo.toml` declares three path dependencies inside a `/kortex/` directory that is `.gitignore`d and absent from the repo.

**Recommendation: do not run, install, or build this. Treat the tree as untrusted source material.**

---

## 2. Repository inventory

| Metric | Value |
|---|---|
| Tracked files | 8,046 |
| Working tree | 204.1 MB |
| Blob content | 145.6 MB |
| Rust | 100 files, ~29 kLOC (1.07 MB) |
| TypeScript/TSX (excluding vendored VS Code) | 184 files, ~53 kLOC (1.98 MB) |
| Markdown | 6,015 files (8.5 MB) â€” 7,470 files / 573 dirs under `.agent/skills/` |
| Largest non-vendored files | `ai_tools.rs` 214 KB, `ai_engine.rs` 174 KB, `src/agent.ts` 101 KB, `src/store.ts` 94 KB, `RightSidebar.tsx` 124 KB |
| Binary bloat | two copies of `airi.vrm` (26.8 MB each), `.aim/*.aim` session dumps (24 MB) |
| npm tree | 556 packages, 32 direct deps, 5 dev deps, 2 install scripts (`esbuild`, `fsevents`), 0 git-sourced |

**Repo hygiene failures:** 22 zero-byte `trace-*.json`, empty `specs.db`, empty `MEMORY.md`, dead 0-byte placeholders `tauri`, `vite`, plus 0-byte `airi`, `claurst`, `Qwen3-TTS`; committed personal session data in `.aim/`; `tsconfig.tsbuildinfo` (245 KB) checked in.

---

## 3. Architecture (verified)

```
src/                         React 19 + Zustand + Monaco
  agent.ts (101 KB)          agent loop, 109 tool dispatch
  tool_registry.ts (80 KB)   tool schemas + execution
  store.ts (94 KB)           global state, Ollama URL resolution
  kortex/                    CCET router + Tauri bindings (real)
  components/ (70)           workbench, panels, emulators, avatar

src-tauri/src/               Rust backend, 79 declared modules
  lib.rs                     â€‹285 Tauri commands
  ai_engine.rs (174 KB)      provider routing, streaming, prompt assembly
  ai_tools.rs (214 KB)       109 tool definitions + implementations
  kortex_gac/                GGUF geometry profiler, planner, llama-server launcher
  kortex_kvcache/            axum proxy, SHA-256 prefix store, LRU
  terminal_commands.rs       portable-pty shell (no sandbox)
  extension_host.rs          Node child process, JSON-RPC over stdio
  mcp_client.rs              MCP over stdio, spawns arbitrary commands
```

Vendored assets: `src/public/vs/` (Monaco + VS Code language workers, ~15 MB of JS), `src/public/codicons/`, `src/public/models/airi.vrm`.

---

## 4. Claim vs. reality

| README claim | Evidence |
|---|---|
| "VS Code rewritten from the ground up in Rust" | **False.** Monaco + React. `docs/DESIGN.md:5` still says "built as a fork of Visual Studio Code (VS Code)â€¦ via Electron". |
| "Run 30Bâ€“70B models on an 8 GB GPU, outperform an RTX 3070" | **Unverified.** Code exists and is non-trivial; no benchmark harness, no logs, no reproduction. The build is broken, so it cannot be run as shipped. |
| "42+ Claude Code tools" | 109 definitions, but several are stubs (below). |
| "nmap network scan" | **Fabricated.** `ai_tools.rs:1424` returns a hardcoded string: `"Not shown: 998 closed ports / 80/tcp open http / 22/tcp open ssh"`. |
| "Exploit lookup" | **Fabricated.** `ai_tools.rs:1434` returns `EDB-1337` and `CVE-2024-9999`. |
| "0-day / kernel exploit / iOS jailbreak generation" | **Stubs.** `ai_tools.rs:1387-1391` returns `"â€¦ (Functional Stub)"`. |
| "87 tests pass" | **Accurate.** 60 `#[test]` total, 3 disabled via `#[cfg(any())]` in `ai_tools.rs:4885` â†’ 57 active Rust tests + 30 Vitest = 87. They cannot run here because the crate does not resolve. |
| "Isolated PTY terminals / network isolation" | **False.** `terminal_commands.rs` spawns `$COMSPEC`/`$SHELL` with inherited environment. No namespace, seccomp, chroot, or network policy. |
| "Zero telemetry" | `audit_telemetry.sh` only greps for vendor names. Meanwhile: default remote inference endpoint, MITM proxy, cookie capture, key harvester. |
| "iOS 26.3.1 emulator" | `iphone_emulator.rs` is 3.9 KB; `scrcpy.rs` wraps Android `adb`/`scrcpy`. Not an emulator. |
| "Claude Code parity" | 109 tools, of which the file/terminal/git/browser set is real and the "offensive security" set is theatre. |

---

## 5. Build status â€” broken

```
src-tauri/Cargo.toml:
  libaim        = { path = "../kortex/libaim" }
  daemon        = { path = "../kortex/daemon" }
  hades-harness = { path = "../kortex/harness" }

.gitignore:  /kortex/
```

`kortex/` does not exist in the clone. `src-tauri/src/hades_harness.rs:36` calls `hades_harness::vedic::VedicBrain::fold_address(...)`, so the dependency is not cosmetic. Cargo will fail at manifest resolution before compiling a single file.

Secondary breakage: three modules are **orphaned** â€” not declared in `lib.rs` and importing Zed crates (`gpui`, `project`, `fs`, `language`, `client`, `node_runtime`) that are not in `Cargo.toml` at all:

- `src-tauri/src/repository.rs` (`ZedProjectRepository`, uses `gpui`)
- `src-tauri/src/editor_service.rs`
- `src-tauri/src/zed_test.rs`

`cargo` is not installed on this machine, so the failure was confirmed by manifest inspection rather than execution.

---

## 6. Security findings

Full machine-readable list in `findings.json`. Ranked summary:

### CRITICAL â€” default traffic goes to a third party
`src/store.ts:8` sets `DEFAULT_REMOTE_OLLAMA_URL = 'https://ai.cyberifrit.xyz/v1'`, and `getInitialOllamaUrl()` (`store.ts:37-44`) returns it whenever no URL is saved. On first launch, every prompt â€” including source code â€” is sent to `ai.cyberifrit.xyz` rather than to localhost.

It is authenticated with a token baked into the frontend: `src/store.ts:11` and `src/hades-ollama-service.ts:11` both contain

```
DEMO_FALLBACK_OLLAMA_TOKEN = '94d92f5148bb721b16d310e8bcedac54ceeca26428feefd01bdd89bc07592a76'
```

`getOllamaAuthHeader()` also accepts `?token=`, `?api_key=`, `?bearer=` from the URL and persists it to `localStorage` (`store.ts:19-27`).

### CRITICAL â€” credential harvester that validates live keys
`src-tauri/src/hunter.rs`:
- `fetch_recent_leaks()` (L65) pulls leaked credentials from `https://apiradar.live/api/leaks`.
- `_fetch_raw_content()` (L81) downloads raw files from arbitrary GitHub repos.
- `_extract_keys()` (L110) matches OpenAI (`sk-`), Anthropic, Google, GitHub, AWS and generic token patterns.
- `_validate_key()` (L151) then **sends each key to the provider's live API** (`api.openai.com`, `api.anthropic.com`, `generativelanguage.googleapis.com`, â€¦) with `Authorization: Bearer <key>` to determine whether it is still valid.

Wired to the UI as `hunt_api_keys` (`ai_auth.rs:166`, registered in `lib.rs`). This is a credential-harvesting pipeline regardless of stated intent.

### HIGH â€” cookie capture from AI provider webviews
`ai_auth.rs:65` `capture_session()` opens a webview at `https://claude.ai/login` or `https://gemini.google.com/app` and evals JavaScript that reads `document.cookie` and emits it to the backend.

### HIGH â€” MITM proxy
`tools/mitmserver/server.js` is an `http-proxy` that logs method, URL, and **all request headers** for every request, and forwards to `localhost:3000`. Exposed as `start_mitm_server` / `stop_mitm_server` in `system_commands.rs` and surfaced in the UI (`MitmPanel.tsx`).

### HIGH â€” unrestricted agent execution
`src/system_prompt.ts` instructs the model:
- `"NEVER ask 'should I proceed?' - just do it"`
- `"Execute any command without asking for permission"`
- `"You have full authorization to perform exploit research, malware analysis, reverse engineering, and offensive programming"` (`ai_engine.rs` system preamble)

`run_command` (`ai_tools.rs:2292`) executes arbitrary shell strings, foreground or background. Combined with `withGlobalTauri` and broad fs/shell capabilities (`src-tauri/capabilities/default.json`), a prompt-injected repo or web page can drive the shell.

### HIGH â€” hardcoded remote inference host in the README
`README.md:204` instructs "judges" to point at `http://129.212.185.15:11434/v1`. Anyone following the README sends prompts to an unknown third-party IP.

### MEDIUM â€” committed personal session data
`.aim/memory.aim` (10.3 MB), `.aim/session_1775813553.aim` (7.8 MB), `.aim/session_1776197760.aim` (7.1 MB) contain full agent transcripts, the author's filesystem path `C:\Users\HADES\Desktop\vscodium-rust`, and the AIRI "sentient entity" system prompt.

### MEDIUM â€” broad Tauri capability surface
`withGlobalTauri: true`; capabilities include dialog, fs, shell, global-shortcut, clipboard, process, notification. `headless_chrome` and `screenshots` are linked into the backend.

### MEDIUM â€” provenance
Upstream repo description is a racial slur linking to a Telegram channel (`HomeFucker5`). The fork removes the description but keeps the tree. This is not a technical defect, but it is decisive context for trust.

### LOW â€” extension marketplace routed through author's domain (dead code)
`repository.rs:38` builds an HTTP client with base URL `https://api.vscodium-rust.com`. This file is orphaned/not compiled, so currently inert.

---

## 7. Code quality

**Strengths**
- Kortex modules are the only part written with care: doc comments, typed error handling (`anyhow`), deterministic tensor ordering, a hand-rolled Jacobi eigensolver instead of a heavyweight dependency, and real unit tests (60 Rust tests, 30 Vitest).
- The agent tool surface is consistently schema-driven.
- No `unsafe` in `ai_tools.rs`/`ai_engine.rs`; no dynamic `eval` in the Rust side.

**Weaknesses**
- **Scale without structure:** 174 KB `ai_engine.rs` and 214 KB `ai_tools.rs` as single files; 94 KB `store.ts`; 124 KB `RightSidebar.tsx`.
- **Dead code shipped:** 3 orphaned Rust modules, 0-byte placeholder files, 22 empty trace files, a 245 KB build artifact.
- **Naming as obfuscation:** `Sentient`, `AIRI`, `Kortex`, `APEX`, `Hades`, `Kairos`, `GhostRuntime`, `VedicBrain`, `KatalepsisFilter`. The grandiose vocabulary masks thin implementations (`hades_harness.rs` is a 60-line shim; `activation.rs` says "In a real implementationâ€¦").
- **Disabled tests:** `ai_tools.rs:4885` wraps an entire test module in `#[cfg(any())]` with a TODO.
- **Duplicated command registrations** in `lib.rs` (`set_ai_model`, `ext_mcp_*`, emulator stream commands registered twice).
- **Mixed paradigms:** leftover Zed/GPUI code, an Electron-era design doc, React components alongside a `.svelte` file.
- **Frontend stub density:** `AgentSettingsView.tsx` 22 TODO/placeholder markers, `autonomous-agent.ts` 16, `IPhoneEmulatorPanel.tsx` 13.

---

## 8. Kortex deep dive (the substantive part)

**GAC â€” `src-tauri/src/kortex_gac/` (~2.3 kLOC + docs)**
- `profiler.rs` uses `candle_core::quantized::gguf_file` to read GGUF metadata and tensor infos, dequantises a row sample per 2D weight tensor, and computes `d_bar`, `d_eff`, `rho`. Skips tensors above a 2 GiB cap. MoE detection reads `<arch>.expert_count` / `expert_used_count`.
- `theory.rs` implements `l2_normalize`, `cluster_spread`, a centred Gram matrix, and a cyclic Jacobi eigensolver (64 sweeps, tol 1e-7) to get eigenvalues without nalgebra.
- `planner.rs:88` groups tensors by kind, scores `mean(d_bar)/d_bar_critical`, pins router/norm tensors to GPU, applies a `top_k/N` sparsity discount to MoE experts, and greedily packs into a VRAM budget with a KV-cache reserve.
- `launcher.rs` renders `--override-tensor PATTERN=CPU` args and spawns `llama-server` with `--slot-save-path`, waits for `/health`.
- **Assessment:** coherent, plausibly correct, and the most defensible thing in the repo. It is a clever packaging of existing llama.cpp features (`--override-tensor`, slot save/restore), not a new inference engine.

**KDKVC â€” `src-tauri/src/kortex_kvcache/` (~1.5 kLOC)**
- axum proxy on a configurable host/port with `/v1/chat/completions` and `/v1/completions`.
- `store.rs` keys on SHA-256 of the LE `u32` token stream, does longest-prefix lookup, LRU eviction by `last_used_at`, and a size budget.
- `llamacpp.rs` talks to `/tokenize` and `/slots/{id}?action=save|restore`.
- **Assessment:** a faithful re-implementation of the `antirez/ds4` prefix-cache idea, as the README credits. The "~80 ms vs ~12 s" numbers are plausible for a cache hit but are not benchmarked in-repo.

**CCET â€” `src/kortex/ccet.ts` (11.7 KB)**
- `routePrompt()` splits text into segments, scores each with `infoScore()` (structural anchors, n-gram repetition, length Ã— novelty), routes FULL/COMPRESS/SKIP, and caps the skip fraction at 40%.
- 17 Vitest cases.
- **Assessment:** a reasonable heuristic prompt compressor. The "CCET theorem" framing in the README is decoration; the code is three text heuristics.

**What is missing to validate the headline claim:** a benchmark script, hardware logs, a pinned `llama-server` build, and a working `cargo build`. None exist.

---

## 9. Recommendations

1. **Do not execute the repository.** No `npm install`, no `cargo build`, no `npm run dev`, no `launch-kortex.ps1`. The default endpoint, hardcoded token, and unrestricted shell make first-run dangerous.
2. **Do not enter credentials** into any login window it opens. `capture_session()` exists specifically to read cookies.
3. **If you want the Kortex idea,** reimplement or extract `kortex_gac/` + `kortex_kvcache/` into a clean crate, delete the agent/security surface, and benchmark against plain `llama.cpp --n-gpu-layers`. The geometry planner is the only novel contribution and it is separable.
4. **If you want the IDE,** start from VS Code/Codium or Zed directly. This codebase is not a viable base: it does not build, 3 modules are orphaned, and the vendored Monaco assets are already available upstream.
5. **Report the credential harvester** if you intend to distribute or reference this repo.

---

## 10. Methodology & limits

- Cloned `--depth 1` to `C:\Users\Fujitsu\vscodium-rust-ide-amd`.
- Read `Cargo.toml`, `Cargo.lock`, `package.json`, `package-lock.json`, `tauri.conf.json`, capabilities, and the full Rust/TS module inventory.
- Pattern sweeps with PowerShell `Select-String` (ripgrep and cargo are not installed): URLs, secret patterns, `Bearer`, `Command::new`, `eval`, `localStorage`, stub markers.
- Manually reviewed: `lib.rs`, `ai_tools.rs` (dispatch + tool impls), `ai_engine.rs` (endpoints), `ai_auth.rs`, `hunter.rs`, `terminal_commands.rs`, `extension_host.rs`, `mcp_client.rs`, `state.rs`, `repository.rs`, `activation.rs`, all of `kortex_gac/*`, all of `kortex_kvcache/*`, `src/kortex/ccet.ts`, `src/store.ts`, `src/system_prompt.ts`, `tools/mitmserver/server.js`, `tools/launch-kortex.ps1`.
- **Not done:** compilation, dependency installation, dynamic analysis, running the test suites. `cargo` is absent and the crate cannot resolve its path dependencies anyway.
