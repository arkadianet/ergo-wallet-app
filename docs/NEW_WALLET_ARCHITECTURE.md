# New Cross-Platform Ergo Wallet — Architecture Research

**Date:** 2026-08-14  
**Author:** arkadianet (research agent)  
**Status:** Decision-ready. No production code.  
**Workspace:** `arkadianet/ergo-wallet-app` (fork of `ergoplatform/ergo-wallet-app`, branch `develop`, version `2.4.2313`)

This report answers: how to build a **new** custody wallet with a **Rust core** (crypto, HD derivation, tx building, signing, DeFi protocol logic), **Android first**, designed to also run on iOS and desktop.

---

## Sources actually read

| Source | Location used | Notes |
|--------|---------------|--------|
| Official wallet (incumbent) | `/workspace` (this repo) | Kotlin/JVM, ergo-appkit SNAPSHOT |
| sigma-rust / ergo-lib 0.28 | cloned `https://github.com/ergoplatform/sigma-rust` (develop) | Official Ergo Rust library |
| arkadianet/ergo 0.5.3 | cloned `https://github.com/arkadianet/ergo` | From-scratch Rust full node |
| arkadianet/citadel 0.2.4 | cloned `https://github.com/arkadianet/citadel` | Tauri v2 desktop DeFi |
| kadia-wallet-app | **not found** | Not on this machine; `github.com/arkadianet/kadia-wallet-app` 404; no public GitHub match |
| flutter_rust_bridge | docs + crates.io | Stable **2.12.0** (2026-03-29); 2.13.0-beta.6 on 2026-08-08 |
| Flutter | docs.flutter.dev | Stable **3.47** (2026-08-12) |
| Compose Multiplatform | JetBrains blog | iOS stable in **1.8.0** (2025-05); **1.11.0** (2026-05) |
| Tauri v2 | v2.tauri.app | Mobile exists; team walked back “first-class mobile” |
| EIPs | ergoplatform/eips | EIP-0019 / EIP-0020 / EIP-0043 |

Local paths from the research prompt (`~/coding/arkadianet/...`, `~/coding/reference/...`, `~/coding/archive/kadia-wallet-app`) **do not exist on this Cloud Agent VM**. Claims about kadia’s `wallet-core` / `wallet-ffi` / build scripts could not be verified from source.

---

## Fact-check of “already established” claims

| # | Claim | Verdict |
|---|--------|---------|
| 1 | Official wallet is a Kotlin/JVM **KMP** app | **Wrong wording.** Shared code is plain JVM library modules (`:common-jvm`, `:common-compose`, `:sqldelight`). There is **no** `kotlin-multiplatform` plugin, no `expect`/`actual`. iOS is **RoboVM 2.3.19** + UIKit, not KMP/Native. Android persistence is **Room 2.4.3**, not SQLDelight (SQLDelight is desktop/iOS only). |
| 1 | Legacy ergo-appkit, Fragments + ViewBinding, partial Compose | **Correct.** `ergo-appkit_2.11:44fddd97-SNAPSHOT` in `common-jvm/build.gradle`. Kotlin **1.6.10**, Compose **1.1.0**. ~47 Fragment classes vs 8 Android `@Composable`s. Desktop is fully Compose + Decompose. |
| 1 | Signing centralized in `uilogic/transactions/` + `ErgoFacade.kt` | **Mostly correct.** Add `ColdWalletSigningUiLogic` (separate from `SubmitTransactionUiLogic`) and `ErgoAuthUiLogic` (`ErgoFacade.signMessage`). Platform UI owns auth prompts. |
| 2 | sigma-rust = ergo-lib 0.28, interpreter, small WIP compiler, bindings, not a full node, has reduced-tx | **Correct**, with numbering fix below. `ergo-lib` crate-type is `["cdylib", "rlib"]`. `ergoscript-compiler` is ~2.5k LOC, optional feature, still listed as roadmap #370. JNI Android bindings are a **scaffold** (2 functions), not a full wallet API. iOS + C bindings are mature. |
| 2 | “EIP-20 reduced-tx support (`chain/transaction/reduced.rs`)” | **Wrong EIP number.** `reduced.rs` cites **EIP-19** (`eip-0019.md`). ErgoPay is **EIP-0020** and *carries* a ReducedTransaction. A later **EIP-0043** also titled “Reduced Transaction” exists. See §C. |
| 3 | arkadianet/ergo deeper than sigma-rust; ergo-wallet strong; no network client; no mobile bindings; no reduced-tx; pre-1.0 “not for funds custody” | **Correct.** Workspace version **0.5.3**, Rust 1.95.0. `SECURITY.md` and `README.md` both forbid funds custody. Zero `cdylib`/`jni`/`android`/`uniffi`. Zero `ReducedTransaction`. `ergo-wallet` has no HTTP deps. |
| 3 | LOC: compiler ~37k, sigma ~29k, ser ~19.5k | **Correct** (`wc -l` on `*/src`: compiler 37,063; sigma 28,885; ser 19,548). **Correction:** `ergo-compiler` is **not** a workspace member — it is orphaned from the default build. `ergo-state` is ~35k (codemap said ~33k). |
| 4 | Citadel protocols + ergo-tx + ergopay-core | **Correct**, with deltas: workspace **0.2.4**; `ergo-lib` pinned to sigma-rust **`develop`** (0.28 lineage), not crates.io; **no `crates/protocols/rosen`** in this tree; `citadel-timelock` is design-only; Citadel **does not hold keys** (Nautilus / ErgoPay handoff). |
| 5 | kadia Flutter + Rust wallet | **Unverifiable here.** Treat as claimed prior art only. |
| 6 | Local machine already set up for Rust-on-Android | **Unverifiable on this VM.** No `~/Android/Sdk`, no Flutter install, no `cargo-ndk` in this environment. |
| 7 | Pure-Rust UI not production-mature; three realistic shells; tentative Flutter + FRB | **Still the right call.** CMP iOS is now *actually* production-stable (1.8+, 2025-05), so CMP is a closer second than that conclusion assumed. Tauri mobile is still the wrong shell for a custody wallet. |

---

## A. UI shell comparison (2026)

The established conclusion stands: **do not ship a funds-holding wallet in egui / Iced / Slint / Dioxus / GPUI.** Those are desktop/web-first. “Cross-platform Rust wallet” means **Rust core + a mature UI shell**.

### A.1 Flutter (recommended)

**Maturity (2026).** Flutter stable **3.47** shipped 2026-08-12 ([docs.flutter.dev/release/whats-new](https://docs.flutter.dev/release/whats-new)). 2026 cadence is quarterly (3.41 Feb, 3.44 May, 3.47 Aug). Impeller is the default renderer on iOS and Android since 3.29. Flutter 3.38 added NDK r28 / 16 KB page-size support required by Google Play (Android 15+). iOS 26 / Xcode 26 / UIScene lifecycle are supported as of 3.38.

**Android + iOS.** First-class, store-proven for a decade. Desktop (Windows / macOS / Linux) is the same toolkit — later expansion is not a rewrite.

**Ecosystem.** pub.dev is the strongest mobile plugin market of the three options. For a wallet the relevant packages exist and are maintained:

| Need | Package | Backing |
|------|---------|---------|
| Biometrics prompt | `local_auth` | Android `BiometricPrompt`, iOS `LocalAuthentication` |
| Encrypted secret store | `flutter_secure_storage` | Android Keystore (RSA-OAEP+AES-GCM or AES-GCM+biometrics), iOS Keychain |
| Hardware-bound unlock | `biometric_security` (2026) | Keystore / Secure Enclave *use* as the gate, not a boolean |
| Background sync | `workmanager` | Android WorkManager; iOS BGTaskScheduler (weaker) |
| Notifications | `flutter_local_notifications` | Both platforms |
| QR | `mobile_scanner` / `qr_flutter` | Camera + render |
| FFI | `flutter_rust_bridge` 2.12 | Official **Flutter Favorite** |

**Finance-app widgets / theme.** Material 3 is default (since 3.16); 3.35 finished component-theme normalization. Cupertino library is first-party for iOS-native controls. A custody wallet does not need a exotic widget kit — lists, amount fields, confirmation sheets, QR, biometric sheets are all stock. Theme quality is high enough for a serious finance app if you design tokens deliberately (do not ship default purple Material).

**Biometrics / secure enclave.** Do **not** treat `local_auth` returning `true` as a security boundary (trivial to hook). The correct pattern, which the official wallet already approximates in Kotlin, is: wrap the mnemonic with a Keystore/Keychain key that has `setUserAuthenticationRequired(true)` / `SecAccessControl` user-presence. `flutter_secure_storage`’s `AndroidOptions.biometric(enforceBiometrics: true)` and iOS `KeychainAccessibility.first_unlock_this_device` implement this. Prefer `ThisDeviceOnly` so iCloud backup does not exfiltrate the blob.

**Background sync / notifications.** Android: WorkManager via `workmanager`, same idea as the incumbent’s `android/.../BackgroundSync.kt` (`PeriodicWorkRequest`, interval from prefs). iOS background fetch is less reliable — design sync as “on foreground + optional BGTask”, not “always-on node”.

**Release size.** Typical Flutter release APK is ~15–25 MB. A release-optimized Rust `cdylib` (`opt-level = "z"`, LTO — already the sigma-rust workspace profile) adds roughly 3–8 MB per ABI. Ship `arm64-v8a` + `x86_64` (emulator) for debug; Play App Bundle splits ABIs. Comparable to or smaller than the current minified Kotlin/Scala appkit APK.

**Developer experience.** Hot reload, Widget Previews (stable as of 3.47), one UI codebase for Android/iOS/desktop. FRB codegen is a one-liner (`flutter_rust_bridge_codegen create`). The cost is a Dart team skill and a JNI/NDK build pipeline — claimed to already exist from kadia, unverified here.

**Risks.** Dart is a second language. Plugin quality varies — pin and audit anything that touches secrets. iOS background work is weaker than Android.

### A.2 Compose Multiplatform (close second — not the pick)

**Maturity (2026).** This is the biggest change vs the tentative recommendation. JetBrains declared **Compose for iOS stable and production-ready in CMP 1.8.0** (2025-05-06): feature parity for popular cases, type-safe navigation + deep links, VoiceOver / AssistiveTouch, ~9 MB iOS size delta vs SwiftUI, startup/scroll claimed comparable to native ([JetBrains blog](https://blog.jetbrains.com/kotlin/2025/05/compose-multiplatform-1-8-0-released-compose-multiplatform-for-ios-is-stable-and-production-ready/)). **1.11.0** (2026-05) made concurrent iOS rendering the default and added experimental native `UIView` text input. 1.10.0 (2026-01) brought unified `@Preview` and Navigation 3 on non-Android targets.

**Android + iOS.** Android is native Jetpack Compose (best-in-class). iOS is now a credible production target, not a beta science project. Desktop Compose is also first-party (the incumbent desktop app already uses JetBrains Compose 1.1.0 — ancient, but the *idea* is proven in-tree).

**Ecosystem.** Growing (`klibs.io`). Weaker than Flutter for camera/QR/biometrics/secure-storage *as shared multiplatform libraries* — you often write `expect`/`actual` against AndroidX Biometric + iOS Keychain. That is fine for a Kotlin shop; it is extra work for a Rust shop.

**Finance-app widgets.** Material 3 on Android is excellent. iOS Material-looking apps can feel off — budget time for Cupertino-ish tokens or accept a Material look on iOS.

**Biometrics / Keystore.** Best *native* interop of the three (you are already on the JVM/ObjC boundary). The incumbent’s `AndroidEncryptionManager` / `IosEncryptionManager` could be ported almost directly. This is CMP’s strongest argument.

**Background sync.** AndroidX WorkManager is native. iOS still needs a native worker. Same as Flutter, but you write Kotlin/Swift rather than a plugin.

**Release size.** CMP iOS +~9 MB vs SwiftUI (JetBrains). Android similar to a Compose app. Adding a Rust `.so` via JNI is the same size cost as Flutter.

**Developer experience.** Hot Reload exists. If the team were Kotlin-first and the core were appkit, CMP would win. **The core is Rust and the DeFi logic already lives in Citadel’s Rust crates.** CMP forces a JNI/UniFFI bridge that sigma-rust has not built (see §B). The incumbent’s Kotlin 1.6 / Compose 1.1 stack is also *not* a starting point — a CMP app would be a greenfield Kotlin 2.1+ project, not an upgrade of this repo.

**Verdict.** Viable in 2026. Choose it only if you decide the UI language should be Kotlin and you are willing to invest in UniFFI/JNI. Do not choose it to “reuse the official wallet UI” — that UI is Fragments, not modern CMP.

### A.3 Tauri v2 mobile (reject for custody)

**Maturity.** Tauri 2.0 shipped mobile as a foundation, then explicitly walked back “mobile as a first-class citizen” in the 2.0 RC notes: desktop features/plugins are not all ported; they do not want that expectation ([v2.tauri.app/blog/tauri-2-0-0-release-candidate](https://v2.tauri.app/blog/tauri-2-0-0-release-candidate/)). CLI has `tauri android` / `tauri ios` (init/dev/build/run). Official plugins include **biometric** (Android + iOS). Keystore is a *community* plugin (`impierce/tauri-plugin-keystore`), not official.

**Citadel evidence.** `app/Cargo.toml` uses `tauri = { version = "2", features = [] }`, `@tauri-apps/cli ^2.9.6`. `app/tauri.conf.json` is a single 1400×900 desktop window. **No iOS/Android bundle targets.** Citadel’s “mobile” is QR-handoff to an *external* ErgoPay wallet, not a Tauri mobile app.

**Why not for a custody wallet.**

1. UI is a **WebView**. Finance UX, secure text fields, screenshot flags, and “is this the real app” trust are worse than Flutter/CMP.
2. Secrets crossing JS ↔ Rust IPC is a larger attack surface than FRB/JNI.
3. Background sync / notifications / biometric-bound Keystore are plugin-shaped and uneven on mobile.
4. You already have Citadel as the Tauri desktop DeFi client. A second Tauri app that *also* holds keys collapses two threat models into one.

**Keep Tauri where it is:** Citadel desktop, no keys, Nautilus/ErgoPay signing.

### A.4 Recommendation

**Flutter + flutter_rust_bridge 2.12 (or current 2.13 beta once stable) + Rust `cdylib`.**

Reasoning, in order:

1. Android-first with a real iOS/desktop path from day one, without RoboVM or a WebView.
2. The valuable existing code (Citadel protocols, ergo-lib, claimed kadia FFI) is **Rust**. Flutter is the shell that talks to Rust most smoothly in 2026 (FRB is Flutter Favorite; CMP’s Rust story is DIY).
3. Wallet-grade plugins (Keystore/Keychain, biometrics, QR, WorkManager) are boring and solved.
4. CMP is now good enough that it is a *reversible* decision for the first month — but starting CMP means writing the bridge twice if you later want Flutter, while starting Flutter does not block a later Kotlin desktop experiment.
5. Do not inherit the official wallet’s Fragment/RoboVM UI. Fresh UX. Copy *flows*, not widgets.

---

## B. Rust ↔ shell bridging

### B.1 flutter_rust_bridge 2.12 (recommended)

Primary sources: [fzyzcjy.github.io/flutter_rust_bridge](https://fzyzcjy.github.io/flutter_rust_bridge/), crates.io `flutter_rust_bridge` **2.12.0** (2026-03-29), **2.13.0-beta.6** (2026-08-08).

**What it generates.** You write ordinary Rust (`async fn`, structs, enums, `Result`, streams, opaque types, `&mut`, traits). Codegen emits Dart + glue. Folder-of-files input (not a single `api.rs`). Dart calls Rust as if it were Dart; Rust can call Dart.

**Ergonomics.** Highest of the three. One-liner project integrate. Official Flutter Favorite (first batch of 7 at reboot).

**Async.** All four combinations: sync/async Rust × sync/async Dart. CPU work on a thread pool; IO on Tokio (`rust-async` feature). Streams via `StreamSink` — the right shape for wallet sync progress.

**Memory / ownership.** Default path is copy/serialize (SSE codec, “several times faster under some workloads”). Opaque Rust types are `Arc`-backed handles — **this is how a `Wallet` / `UnlockedSession` should cross the boundary**, not as a JSON mnemonic string. Zero-copy for big arrays. Auto locking; sanitizer CI (ASAN/MSAN/LSAN, Valgrind).

**Large API surface.** Codegen scales to dozens of wallet + DeFi operations. Citadel’s Tauri layer already has ~118 commands; FRB will generate the equivalent from Rust modules (`api/wallet/`, `api/tx/`, `api/defi/amm/`, …). Do **not** flatten everything into JSON strings (kadia’s claimed approach).

**Cost.** Codegen in CI; NDK/cargo-ndk per ABI; you must keep the public Rust API FFI-friendly (no `ergo-lib` types leaking — wrap in DTOs / opaque handles).

### B.2 Hand-rolled `DynamicLibrary` / C ABI (kadia’s claimed approach)

**What kadia is claimed to have done.** `rust/wallet-ffi` as `cdylib`/`staticlib`, `#[no_mangle]` C ABI, JSON-string marshalling, Dart `RustWalletBridge` loading the `.so` manually, `scripts/build_rust_mobile.sh` + `scripts/sync_android_jni_libs.sh` for `aarch64` / `x86_64` / `armv7`.

**Cannot verify.** Repo not present. The *pattern* is well-known and we can evaluate it.

**Ergonomics.** Poor at scale. Every new operation is a `extern "C" fn`, a Dart `lookupFunction`, a JSON schema, and a manual error channel. This is how kadia (if the claim is right) and how `ergo-lib-c` (~303 `extern "C"` functions) work.

**Async.** You invent it (ports, worker threads, or “call sync on a Dart isolate”). Easy to block the UI isolate or leak sessions.

**Memory.** JSON strings mean the mnemonic *will* appear as a Dart `String` unless you are extremely careful. Handles as raw pointers without codegen are use-after-free magnets.

**Scaling.** Fine for 10 functions. Painful for “dozens of wallet + DeFi operations.” Citadel already proved the API will be large.

**When to use.** A **one-week spike** (compile `ergo-lib` or `ergo-wallet` to `libwallet.so`, call one `extern "C"` from Dart) to de-risk NDK. Do not ship the product on this.

**Related existing C ABI.** `sigma-rust/bindings/ergo-lib-c` 0.28.0 (`crate-type = ["staticlib"]`) already exposes `ergo_lib_wallet_from_mnemonic`, `ergo_lib_wallet_sign_transaction`, `ergo_lib_wallet_sign_reduced_transaction`, plus reduced-tx helpers. That is a better hand-rolled baseline than inventing a new C API — but FRB still wins for the Flutter app.

### B.3 JNI for Compose Multiplatform

**What exists today in sigma-rust.**

```3:10:/tmp/research/sigma-rust/bindings/ergo-lib-jni/src/main/java/org/ergoplatform/wallet/jni/WalletLib.kt
object WalletLib {
    init {
        System.loadLibrary("ergowalletlibjni")
    }
    @JvmStatic external fun addressFromTestNet(addressStr: String): Long
    @JvmStatic external fun addressDelete(address: Long)
}
```

`bindings/ergo-lib-jni/Cargo.toml`: `crate-type = ["staticlib", "cdylib"]`, crate name `ergowalletlibjni`. Gradle uses `org.mozilla.rust-android-gradle` 0.9.2, NDK 26.2, ABIs `arm/arm64/x86/x86_64`, minSdk 21. CI runs `connectedCheck` on API 29. Parent README still lists “Kotlin bindings for Android” as **roadmap #369**.

**Ergonomics.** Writing JNI by hand for a Citadel-sized API is a multi-month project. UniFFI (Mozilla) or `jni` + a thin Kotlin facade is the 2026 default if you pick CMP. UniFFI gives Kotlin + Swift from one Rust IDL — attractive for CMP+iOS — but it is another toolchain, and Citadel/FRB do not use it.

**Async.** JNI is sync. You wrap in coroutines / Dispatchers.Default. No first-class stream story.

**Memory.** `jlong` handles + explicit `delete` (see `addressDelete`) — the C-core pattern. Easy to leak on Kotlin cancellation.

**Scaling.** Worst of the three unless you adopt UniFFI. sigma-rust’s own JNI surface after years is **two functions**.

### B.4 Bridge recommendation

| Criterion | FRB 2.12 | Hand-rolled C / JSON | JNI / UniFFI (CMP) |
|-----------|----------|----------------------|--------------------|
| Ergonomics | Excellent | Poor | Poor (JNI) / good (UniFFI) |
| Async | Native | DIY | Coroutines wrap |
| Ownership | Opaque `Arc` + SSE | You own every free | Handles + delete |
| Large API | Codegen folders | Linear pain | UniFFI IDL or explode |
| Fits recommended shell | Yes | Spike only | Only if CMP wins |

**Ship FRB.** Use a thin C ABI spike only to prove the `.so` links. Do not marshal secrets as JSON strings.

---

## C. Core decision: ergo-lib 0.28 vs arkadianet/ergo

### C.1 EIP numbering (read this before the rest)

The prompt’s “EIP-20 reduced-tx” mixes three documents:

| EIP | What it actually is | Who implements it |
|-----|---------------------|-------------------|
| **EIP-0019** | Cold-wallet protocol **and** the `ReducedTransaction` type sigma-rust cites in `ergo-lib/src/chain/transaction/reduced.rs` | sigma-rust, official wallet (`ColdWalletUtils.kt`), ergo-appkit |
| **EIP-0020** | **ErgoPay** — URL/QR protocol that *transports* a ReducedTransaction (or a URL that returns one) | Official wallet `ErgoPay.kt` / `ErgoPaySigningUiLogic`; Citadel `ergopay-server` + `ergopay-core` |
| **EIP-0043** | Later “Reduced Transaction” data-format EIP (Proposed 2021-08-18) | Overlaps EIP-19 conceptually |

A mobile wallet must implement **both**:

- **Reduce + sign ReducedTransaction** (EIP-19/43) — so ErgoPay dApps and Citadel can hand us bytes.
- **ErgoPay client** (EIP-0020) — fetch `ErgoPaySigningRequest`, show it, sign, optional `replyTo`.

arkadianet/ergo has **neither**. Citadel’s `ergopay-core` converts EIP-12 unsigned JSON → sigma-serialized ReducedTransaction bytes via ergo-lib’s `reduce_tx`, with `reduce_transaction_fallback` (manual EIP-19-style bytes) for HodlCoin trees ergo-lib cannot parse.

### C.2 ergo-lib 0.28 (sigma-rust) — recommended core

**What you get, cited:**

- Version `0.28.0` (`ergo-lib/Cargo.toml`). Changelog date 2024-08-09; this clone’s 0.28 bump is “Use precomputed tables” (k256).
- Wallet: `Wallet::from_mnemonic`, `from_secrets`, `sign_transaction`, `sign_reduced_transaction`, multi-sig commitments (`ergo-lib/src/wallet.rs`).
- HD: BIP-32 / BIP-44 / EIP-3, coin type **429**, path `m/44'/429'/acc'/0/address...` (`derivation_path.rs`). Official wallet uses the same path (`ErgoFacade.kt` `getAddressDerivationPath`).
- Mnemonic: BIP-39, PBKDF2-HMAC-SHA512, 2048 iterations (`mnemonic.rs`). Generation behind feature `mnemonic_gen`.
- Tx builder + box selection + data inputs + context extensions (`tx_builder.rs`).
- REST client behind feature `rest` (`ergo-rest` — node HTTP client, not a node).
- Reduced tx: `ReducedTransaction` + `reduce_tx` + `SigmaSerializable` (`chain/transaction/reduced.rs`).
- Bindings: WASM (Yoroi), C (~303 exports, used by iOS Swift package), Python, JNI scaffold.
- Workspace release profile: `opt-level = "z"`, `lto = true`.

**What it is not.** No consensus validation, no UTXO store, no block sync. `ergo-p2p` / `ergo-rest` roadmaps still list Wasm/iOS/Android bindings as future work. Compiler is optional and incomplete vs Scala.

**Fit for a light wallet.** This is exactly the job: talk to a public node/explorer, build and sign, do not embed a node.

### C.3 arkadianet/ergo 0.5.3 — do not use as mobile core (v1)

**What is genuinely better than sigma-rust:**

- Scala-parity interpreter (`ergo-sigma` 28,885 LOC) and ser (`ergo-ser` 19,548 LOC).
- Consensus crates sigma-rust lacks: `ergo-validation` (16,123), `ergo-state` (35,403).
- `ergo-wallet` (8,895 LOC): BIP39 (`bip39` v2), EIP-3 + pre-EIP-3, Scala-compatible AES-GCM secret storage, `UnsignedTxBuilder` with **`data_inputs` and arbitrary ErgoTree output bytes**, box selectors, Schnorr/DHT/AND-OR-threshold proving, multi-sig hints. **Verified:** `cargo build -p ergo-wallet --release` succeeds without `ergo-node`.
- Mainnet sync-to-tip claimed 2026-04-26 at height 1,771,976.

**What is missing for mobile (concrete):**

| Gap | Evidence | Effort if we insisted |
|-----|----------|------------------------|
| No `cdylib` / JNI / NDK / UniFFI / cbindgen | Workspace grep: zero matches | 1–2 weeks for a thin `wallet-ffi` crate + cargo-ndk per ABI (`aarch64-linux-android`, `x86_64-linux-android`; add `armv7` only if you care about old 32-bit). **Spike this early even if we pick ergo-lib** — same pipeline. |
| No EIP-19 `ReducedTransaction` | Zero matches for `ReducedTransaction` / `eip-20` / `eip19` in `*.rs` | 2–4 weeks to specify, implement, and vector-test against sigma-rust / appkit bytes. **Blocker for ErgoPay.** |
| No network client in `ergo-wallet` | No reqwest/ureq/tokio in that crate | Need a separate light client (explorer + node REST). Citadel’s `ergo-node-client` is ergo-lib-typed — not drop-in. 1–2 weeks to write a reqwest/ureq client, or reuse Citadel’s and stay on ergo-lib. |
| Script gate: only ProveDlog / ProveDHTuple / matured miner-reward | `ergo-wallet` prover docs | DeFi spending (Spectrum, SigmaUSD, Dexy, …) **will not sign**. Lifting the gate needs evaluation context from committed chain state. This is a research project, not a sprint. |
| Transitive `ergo-state` + `redb` | `cargo tree -p ergo-wallet` | Binary-size and mobile-storage problem even if you never open a UTXO DB. Needs a crate split. |
| Alpha / no custody | `README.md` lines 27–32, `SECURITY.md` lines 5–13, 142–149 | Audit + 1.0. Not a schedule you control. |
| `ergo-compiler` orphaned | Not in workspace `members` | Irrelevant for a light wallet that consumes on-chain trees. |

**Relative effort / risk.**

| Path | Effort to Android MVP (HD + send/receive + ErgoPay sign) | Risk |
|------|----------------------------------------------------------|------|
| **ergo-lib 0.28 + Citadel crates + FRB** | Weeks. Bindings and reduced-tx exist. Citadel already reduces EIP-12 → bytes. | Low-medium. Dependency is not “ours”; we track sigma-rust `develop` like Citadel already does. |
| **arkadianet/ergo as signer** | Months. FFI + EIP-19 + network + lift script gate + crate-split `ergo-state` + accept alpha custody disclaimer. | **High.** We would be the first funds-holding client of an alpha node stack. Citadel protocols would need a rewrite off ergo-lib types. |

### C.4 Hybrid (recommended product, not a mash-up)

```
wallet-core  ──depends──►  ergo-lib 0.28 (git pin, same as Citadel)
         └──depends──►  citadel protocol crates + ergo-tx + ergopay-core
         └──does not link──►  arkadianet/ergo
```

Keep arkadianet/ergo as the **node / consensus** project. Revisit a signer swap only after:

1. `ReducedTransaction` byte-compatible with sigma-rust / appkit,
2. a `cdylib` crate in that workspace,
3. script gate lifted for the DeFi trees we ship,
4. an external audit and a custody-policy change in `SECURITY.md`.

A thin “compile `ergo-wallet` to Android `.so`” spike is still worth a day — it de-risks NDK and tells us the size of the redb transitive hit. It is **not** the product core.

---

## D. Security architecture

### D.1 Threat model (custody mobile wallet)

| Asset | Attacker | Goal |
|-------|----------|------|
| BIP-39 mnemonic | Malware, backup sync, screenshots, logs, FRB JSON | Steal funds |
| Unlocked session | Shoulder-surf, rooted hook of `local_auth` | Sign without user |
| Reduced tx from ErgoPay | Malicious dApp | Drain via confusing outputs |
| Node / explorer | Evil or stale API | Wrong balance, fee griefing, not key theft |
| App binary | Sideload / Play impersonation | Phishing |

Non-goals for v1: stealth addresses, MPC, hardware-enclave *signing* (Secure Enclave P-256 ≠ Ergo secp256k1). Hardware is used to **wrap** the seed, not to produce Ergo signatures.

### D.2 What the official wallet does (copy the shape, not the crypto stack)

**At rest.** `WalletConfig.secretStorage` + `encryptionType`:

- `ENC_TYPE_PASSWORD = 1` — `AesEncryptionManager`: PBKDF2-HMAC-SHA1, **65536** iterations, **AES-128-GCM** (`common-jvm/.../AesEncryptionManager.java`).
- `ENC_TYPE_DEVICE = 2` — Android: Keystore AES-GCM key alias `ergowalletkey`, `setUserAuthenticationRequired(true)`, **10 s** validity (`android/.../AndroidEncryptionManager.java`). iOS: 28-char random wrap key in Keychain (`IosEncryptionManager.kt`, `IosKeychainAccess.kt`, `SecAccessControl` user-presence). Desktop: password only.

**In memory.** `SigningSecrets` holds `appkit.SecretString`; `clearMemory()` / `erase()` after sign (`SendFundsUiLogic`, `ColdWalletSigningUiLogic`).

**Unlock.** Platform biometric/password → decrypt blob → `SigningSecrets.fromBytes()` → `ErgoFacade.signSerializedErgoTx` on `ColdErgoClient` → wipe.

**Do not copy.**

- Mnemonic passed through Navigation args as a plaintext string (`SaveWalletFragmentDialog.kt` documents this).
- AES-128 + PBKDF2-SHA1-65536 is below 2026 wallet norms (use AES-256-GCM, PBKDF2-SHA512 ≥ 210k or Argon2id).
- `local_auth`-style “boolean then decrypt” if the Keystore key is not actually `USER_AUTHENTICATION_REQUIRED` on every use (Android’s 10 s window is a usability compromise — prefer per-sign auth for large sends).
- No Rust: all crypto is JVM appkit. A new wallet must not leave the seed in Dart/Kotlin longer than a function call.

### D.3 Recommended split: OS wraps, Rust signs

```
┌─────────────────────────────────────────────────────────┐
│ Flutter shell                                           │
│  • never logs / never stores mnemonic as String         │
│  • flutter_secure_storage holds opaque ciphertext       │
│  • local_auth / biometric_security unlocks Keystore     │
│  • shows tx preview from Rust-decoded ReducedTx         │
└────────────┬──────────────────────────────▲─────────────┘
             │ wrap-key unlock / ciphertext │ balances, tx UI
             ▼                              │
┌─────────────────────────────────────────────────────────┐
│ Rust wallet-core                                        │
│  • generate / import BIP-39 (zeroize)                   │
│  • encrypt seed (Argon2id or PBKDF2-SHA512 + AES-256-GCM)│
│  • HD derive, build, reduce, sign, wipe                 │
│  • DeFi builders (Citadel crates)                       │
│  • explorer/node HTTP (no secrets on the wire)          │
└─────────────────────────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────────────┐
│ OS                                                      │
│  Android Keystore / iOS Keychain (ThisDeviceOnly)       │
│  wraps the Rust ciphertext or the wrap-key              │
└─────────────────────────────────────────────────────────┘
```

**Rules.**

1. **Generate and import in Rust.** Dart receives a `bool` / wallet id, not words. Display words via a short-lived opaque handle + character-by-character UI, then wipe.
2. **Encrypt in Rust** before the blob crosses FFI. Dart/Keystore stores ciphertext (or a wrap-key). If the wrap-key is OS-held, Rust receives it only inside `sign_*`, then `zeroize`.
3. **Sign in Rust.** `Wallet::sign_reduced_transaction` / `sign_transaction`. The shell never sees an extended private key.
4. **Do not keep an unlocked wallet in Dart.** Keep an opaque `UnlockedSession` in Rust with a TTL and an explicit `lock()`.
5. **Password wallets** (no biometrics / desktop): KDF in Rust, not the official wallet’s AES-128/SHA1.
6. **Screenshots / recents:** FLAG_SECURE / iOS analog on seed and sign screens.
7. **Backup:** user writes words. Do not iCloud/Google-backup the Keystore blob (`ThisDeviceOnly` / `setIsStrongBoxBacked` when available).

### D.4 Threat-model tradeoffs

| Design | Pros | Cons |
|--------|------|------|
| Seed only in Rust, file on disk | Simple | Disk backup / root reads the file; no hardware gate |
| Seed only in Keystore as raw extra | OS backup stories | Android Keystore entry size limits; iOS Keychain sync risk; signing still needs the bytes in process |
| **Rust ciphertext + OS wrap-key (pick this)** | Hardware gate; Rust owns crypto; portable to desktop (password wrap) | Two-step unlock; must handle biometric enrollment change (invalidate → re-auth with words) |
| Secure Enclave / StrongBox *signing* | Non-extractable keys | **Cannot** produce Ergo secp256k1 signatures from Apple P-256 enclave keys. Dead end for Ergo. |

Rooted/jailbroken devices: we cannot win. Disclose, optional “block if SafetyNet/Play Integrity / DeviceCheck fails” as a setting, not a fantasy.

---

## E. Reuse the ecosystem — do not rebuild

### E.1 From Citadel (`arkadianet/citadel` 0.2.4) — carry directly

Citadel is **not** a wallet. It builds EIP-12 txs and hands them to Nautilus/ErgoPay. That is ideal: we become the signer it already targets.

**Take (Tier 1 — portable Rust):**

| Crate | Why |
|-------|-----|
| `citadel-core` | Types, errors, fee constants. No ergo-lib, no Tauri. |
| `ergo-tx` | EIP-12 types, box selection, send/burn/UTXO mgmt. Feature `ergo-lib`. `build_send_tx` etc. “Does not manage keys.” |
| Protocol `tx_builder` + calculators + constants | `amm`, `dexy`, `sigmausd`, `lending`, `hodlcoin`, `sigmafi`, `mewlock`, `stake_recovery` |
| `ergopay-core` | `reduce_transaction` / `reduce_transaction_fallback` → ReducedTransaction bytes. Needed so *we* can also *produce* ErgoPay requests (watch-only / desktop companion) and so we share one reducer with Citadel. |
| `ergo-node-client` (optional) | Node REST, EIP-12 UTXOs, `check_transaction` / `submit_transaction`. Tokio. Default URL `127.0.0.1:9053` — mobile needs a public node setting. |

**Do not take:**

| Crate | Why |
|-------|-----|
| `ergopay-server` | Local Axum server for desktop→phone QR. We *are* the phone. |
| `citadel-api` | Tauri façade + ErgoPay orchestration. |
| `app` | Tauri v2 desktop shell. |

**Pinning.** Citadel has **no `Cargo.lock` in the snapshot we cloned.** Workspace pins `ergo-lib` to `github.com/ergoplatform/sigma-rust` branch `develop` plus `[patch.crates-io]`. `lending` declares `ergo-lib = { version = "0.28", features = ["json"] }` while others use workspace git — patch should unify, but **lock a git SHA** in the new wallet. Rosen is **not** in this tree (README/CLAUDE.md: not shipped).

**HodlCoin.** May need `reduce_transaction_fallback` when ergo-lib cannot parse the bank tree — already solved in Citadel.

### E.2 From kadia — unverified, reuse the *idea*

Claimed: `rust/wallet-core` on ergo-lib 0.28 with Spectrum/SigmaUSD/Dexy; `rust/wallet-ffi` C ABI + JSON; Android NDK scripts.

**If the archive is recovered:** steal `scripts/build_rust_mobile.sh` and `sync_android_jni_libs.sh`, not the JSON FFI and not a second copy of AMM/SigmaUSD/Dexy (Citadel’s crates are current and richer: T2T, LP, router, Dexy, lending, …).

**If not recovered:** recreate the NDK pipeline from FRB’s template + cargo-ndk 4.x. Do not block the project on finding kadia.

### E.3 From the official wallet — flows, not UI

**Copy as product requirements / state machines:**

- Create / restore / read-only / cold-wallet split.
- `ErgoPaySigningUiLogic.State` (FETCH_DATA → WAIT_FOR_WALLET/ADDRESS → WAIT_FOR_CONFIRMATION → DONE).
- EIP-19 QR chunking (`QrCodePagesCollector`, `ColdWalletUtils.kt`).
- Explorer + node fallback (`ApiServiceManager`, `preferNodeAsExplorer`).
- Device vs password encryption modes.
- `clearMemory()` discipline.
- Mosaik is optional/later — it is a JVM runtime (`com.github.MrStahlfelge.mosaik`).

**Do not copy:**

- Fragments, ViewBinding, Navigation-args mnemonic, Room/SQLDelight split, RoboVM, Kotlin 1.6, appkit SNAPSHOT, GlobalScope, Compose 1.1.
- The visual language unless you want to look like 2021 Terminus. **Genuinely fresh UX.**

**Do not wrap this repo.** Upgrading it to CMP + JNI + Rust would cost more than a new Flutter app and would still be appkit.

### E.4 From sigma-rust bindings

- Prefer **FRB on `ergo-lib` directly** over wrapping `ergo-lib-c` (double FFI).
- `ergo-lib-ios` is a useful **reference** for what a complete mobile API looks like (Wallet, TxBuilder, ReducedTransaction, RestNodeApi).
- Do not wait for JNI issue #369.

---

## F. Phased delivery

### Phase 0 — De-risk (do this first, before product UI)

Highest-risk unknowns, cheapest tests:

| Spike | Why | Pass criterion |
|-------|-----|----------------|
| **S1.** `ergo-lib` → `aarch64-linux-android` `.so` via cargo-ndk, load from a 20-line Flutter app | NDK / 16 KB pages / LTO size | `Wallet::from_mnemonic` + derive `m/44'/429'/0'/0/0` matches official wallet test vector |
| **S2.** Same `.so` signs an EIP-19 ReducedTransaction from a fixture (Citadel / official-wallet QR dump) | ErgoPay path | Signature verifies on a public node `checkTransaction` |
| **S3.** Optional: compile `arkadianet/ergo` `ergo-wallet` to Android `.so` | Size + “can we ever switch” | Links; report `.so` KB and whether redb comes along. **Do not productize.** |
| **S4.** Keystore wrap: store Rust ciphertext via `flutter_secure_storage` biometric options, unlock, sign, wipe | Security architecture | Enrollment-change fails closed; no mnemonic in `logcat` |
| **S5.** Vendor Citadel `amm` + `sigmausd` + `ergo-tx` + `ergopay-core` into the workspace with a **git SHA** pin | Dependency reality | `cargo test` on those crates in the wallet workspace |

Do **not** start a Fragment rewrite or a Tauri mobile prototype.

### Phase 1 — MVP (Android)

HD wallet, send/receive, signing, ErgoPay.

- Create / restore 12–24 word (official default is **15** — `MNEMONIC_WORDS_COUNT = 15`; also accept 12).
- EIP-3 derivation; detect pre-EIP-3 on restore (official `deprecatedDerivation` flag).
- Receive: address + QR + payment URI.
- Send: ERG + EIP-4 tokens, fee picker, Babel fees *later*.
- Sign reduced + unsigned; broadcast via node; explorer fallback for balances (port `WalletStateSyncManager` behavior, not code).
- ErgoPay client (EIP-0020): static `ergopay:<b64>` and HTTP `ErgoPaySigningRequest`.
- Read-only watch + cold QR (EIP-19) so this app can be the cold side.
- Device-bind or password encryption as in §D.
- Mainnet + testnet flavors (incumbent pattern).

**Out of MVP:** DeFi, Mosaik, ErgoAuth, multi-sig, iOS store release (keep iOS compiling in CI if cheap).

### Phase 2 — DeFi

Vendor Citadel builders behind FRB:

1. Spectrum AMM swap (highest user demand)
2. SigmaUSD mint/redeem
3. Dexy
4. Then lending / HodlCoin / SigmaFi / MewLock / stake recovery as demand says

UI is new. Signing is the Phase 1 path (we sign our own reduced txs — no `ergopay-server`). Optionally stay compatible so Citadel desktop can still ErgoPay-handoff to *this* app.

### Phase 3 — Platform expansion

- iOS store (Flutter; reuse FRB staticlib — `ergo-lib-ios` is a reference, not a dependency).
- Desktop Flutter *or* keep Citadel as the desktop DeFi surface and add a thin Flutter desktop for custody only. **Prefer the latter** — do not kill Citadel.
- ErgoAuth, Mosaik-equivalent (or a simpler dApp browser), multi-sig.

### Highest-risk unknowns (again)

1. **NDK + 16 KB + release size** of ergo-lib + protocol crates — S1.
2. **ReducedTransaction byte compatibility** with official wallet / Nautilus / Citadel — S2.
3. **Secret handling across FRB** (no Dart `String` seed) — S4.
4. **Public node / explorer reliability** without embedding arkadianet/ergo.
5. **sigma-rust `develop` drift** vs crates.io 0.28.0 (Citadel already lives on `develop`).
6. kadia scripts missing — do not treat as a dependency.

---

## Recommended architecture

```
                     ┌──────────────────────────────────────┐
                     │  Flutter 3.47+  (Android first)      │
                     │  Material 3 / Cupertino tokens       │
                     │  secure_storage + biometric_security │
                     │  workmanager, mobile_scanner         │
                     └──────────────────┬───────────────────┘
                                        │ flutter_rust_bridge 2.12
                                        │ (opaque handles, SSE, async)
                     ┌──────────────────▼───────────────────┐
                     │  wallet-ffi  (cdylib / staticlib)    │
                     │  FRB api/ modules only               │
                     └──────────────────┬───────────────────┘
                                        │
                     ┌──────────────────▼───────────────────┐
                     │  wallet-core                         │
                     │  keys, encrypt, hd, sync, sign       │
                     │  ergo-lib 0.28 (git SHA)             │
                     │  ergo-tx, ergopay-core               │
                     │  protocols/{amm,sigmausd,...}        │
                     │  node/explorer HTTP                  │
                     └─────────────┬───────────┬────────────┘
                                   │           │
                    public node    │           │  (not linked)
                    + explorer     │           │
                                   │           ▼
                                   │   arkadianet/ergo  (node project)
                                   │   future signer candidate only
                                   ▼
                            Ergo network
```

Citadel (Tauri desktop) stays a **separate process/product** that can ErgoPay into this wallet.

---

## Concrete crate / module layout

New repo (do not grow `ergo-wallet-app`). Suggested name: `ergo-wallet` / `citadel-wallet` — human decision.

```
wallet/
  app/                          # Flutter project
    lib/
      ui/                       # screens: onboarding, home, send, receive, ergopay, settings
      bridge/                   # generated FRB + thin wrappers
    android/                    # jniLibs filled by script
    ios/
  rust/
    Cargo.toml                  # workspace
    wallet-ffi/                 # cdylib+staticlib, FRB entry
    wallet-core/                # keys, kdf, hd, account, sync, sign
    wallet-net/                 # explorer + node REST (reqwest or ureq)
    vendor/                     # git subtree or path deps
      citadel-core/
      ergo-tx/
      ergopay-core/
      protocols/{amm,sigmausd,dexy,...}
  scripts/
    build_android.sh            # cargo-ndk aarch64, x86_64 [, armv7]
    build_ios.sh
  example.env                   # never commit .env
  docs/
```

`wallet-core` public FRB surface (illustrative):

- `WalletManager.create / restore / lock / unlock / list`
- `AddressService.derive / payment_request`
- `TxService.prepare_send / sign_reduced / sign_unsigned / broadcast`
- `ErgoPayService.parse / fetch / preview`
- `SyncService.refresh` → `StreamSink<SyncEvent>`
- `DefiService.*` (Phase 2)

---

## Dependency list (pin these)

### Rust workspace

| Crate | Version / pin | Role |
|-------|---------------|------|
| `ergo-lib` | git `ergoplatform/sigma-rust` **SHA of develop**, features `std,json,mnemonic_gen,rest` | Chain, wallet, reduce, sign |
| `ergotree-ir` / `ergotree-interpreter` / `sigma-ser` | same git pin (via patch, as Citadel) | Transitive |
| `citadel-core` | path, Citadel 0.2.4 | Types |
| `ergo-tx` | path, feature `ergo-lib` | EIP-12 build |
| `ergopay-core` | path | Reduce to EIP-19 bytes |
| protocol crates | path, as needed | DeFi |
| `flutter_rust_bridge` | **2.12.0** (bump to 2.13 when stable) | FFI codegen |
| `tokio` | 1.40+ (`rt-multi-thread`, `macros`) | Async (Citadel already) |
| `reqwest` | 0.12, `rustls-tls` | Node/explorer (or reuse `ergo-node-client`) |
| `zeroize` | 1 | Wipe secrets |
| `argon2` or `pbkdf2`+`hmac`+`sha2` | current | KDF |
| `aes-gcm` | 0.10 | AES-256-GCM |
| `thiserror` / `serde` / `serde_json` | 1 / 1 / 1 | |
| `bip39` | only if not using ergo-lib mnemonic | Prefer ergo-lib |

**Do not depend on:** `ergo-node`, `ergo-state`, `redb`, `tauri`, `axum` (unless you later embed an ErgoPay *server* on desktop).

### Flutter

| Package | Version (check pub.dev at implement time) | Role |
|---------|-------------------------------------------|------|
| Flutter SDK | **3.47+** | UI |
| `flutter_rust_bridge` | 2.12.x | Match Rust crate |
| `flutter_secure_storage` | latest 10.x | Keystore/Keychain |
| `local_auth` or `biometric_security` | latest | Unlock |
| `workmanager` | latest | Android background sync |
| `flutter_local_notifications` | latest | Optional |
| `mobile_scanner` / `qr_flutter` | latest | QR |
| `freezed` / `riverpod` or `bloc` | team pick | State |

### Toolchain

| Tool | Version |
|------|---------|
| Rust | stable, plus targets `aarch64-linux-android`, `x86_64-linux-android`, later `aarch64-apple-ios` |
| cargo-ndk | 4.1.x (claimed available on the local machine) |
| Android NDK | **r28+** (16 KB pages; Flutter 3.38+ default) |
| minSdk | 26+ (incumbent is 24; 26 simplifies Keystore) |
| JDK | 17 |

---

## Prioritized risk register

| P | Risk | Impact | Mitigation |
|---|------|--------|------------|
| P0 | Seed appears in Dart/logs/screenshots | Total loss | Opaque handles; no JSON mnemonic; FLAG_SECURE; S4 spike |
| P0 | Wrong ReducedTransaction bytes vs ErgoPay ecosystem | Cannot sign dApp txs | S2 against official-wallet + Citadel fixtures; treat EIP-19 as the wire format |
| P0 | Using arkadianet/ergo as signer before 1.0 | Silent consensus/script bugs, funds loss | **Don’t.** Policy already forbids custody. |
| P1 | NDK / 16 KB / `.so` size blow-up when linking protocol crates | Unshippable APK | S1 with LTO; ABI splits; defer DeFi crates until Phase 2 |
| P1 | sigma-rust `develop` breaks Citadel builders | DeFi and signing diverge | SHA pin + `cargo update` ritual; mirror Citadel’s patch |
| P1 | Public node/explorer lying or down | Stuck UX, not key theft | Multi-endpoint, incumbent `preferNodeAsExplorer` pattern |
| P1 | Biometric enrollment change wipes wrap-key | User locked out | Force word-reauth path; document it |
| P2 | FRB codegen vs large DeFi API | Slow iteration | Folder-per-domain; opaque protocol state |
| P2 | iOS background sync weak | Stale balances | Foreground-first; optional notifications |
| P2 | kadia scripts missing | Lost a week of NDK glue | FRB template + cargo-ndk; do not wait |
| P2 | CMP looks tempting mid-project | Split-brain UI | Decide in Phase 0; do not dual-track |
| P3 | Mosaik users miss dApps | Feature gap | Phase 3; ErgoPay covers most signing |
| P3 | Official wallet still exists | Support confusion | Fresh brand/UX; interop via ErgoPay/EIP-19 |

---

## Recommendation in one paragraph

Build a **new** Flutter 3.47+ app whose Rust `wallet-core` depends on **ergo-lib 0.28 (sigma-rust `develop`, SHA-pinned)** plus Citadel’s **`ergo-tx` / `ergopay-core` / protocol `tx_builder`s**, exposed through **flutter_rust_bridge 2.12** as opaque handles — not JSON secrets and not the official Kotlin/appkit/RoboVM tree. Keep **Android Keystore / iOS Keychain as the wrap**, **Rust as the only place that sees the mnemonic and produces signatures**, and keep **Citadel as the keyless Tauri desktop DeFi client** that ErgoPay-handoffs into this wallet. Do **not** put arkadianet/ergo on the custody path until it has EIP-19 byte compatibility, a real FFI crate, a lifted script gate, and a custody-policy change; do **not** use Tauri WebView as the mobile shell; treat Compose Multiplatform as a viable but worse-fitting alternative now that iOS is stable. Phase 0 is five spikes (NDK `.so`, reduced-tx fixture, optional ergo-wallet size probe, Keystore wrap, Citadel vendor pin) before any product UI.

### Three open questions that need a human decision

1. **Brand and repo:** new repository and name (recommended) vs. attempting to evolve `ergo-wallet-app`? Fresh UX is assumed above; confirm you are willing to *not* ship Terminus-lookalike Fragments and to leave this fork as the incumbent.

2. **Custody policy vs. arkadianet/ergo:** is the point of the new wallet to dogfood the from-scratch node stack (accept alpha + months of EIP-19/FFI/script-gate work), or to ship a funds-holding Android wallet this year on ergo-lib (this report’s default)? The two goals conflict.

3. **Desktop story:** Flutter desktop custody app later, or Citadel-only on desktop forever (phone as the only hot wallet, Citadel stays keyless)? This decides whether FRB must stay desktop-clean and whether we ever embed `ergopay-server`.

---

## Appendix: incumbent signing call chain (for interop tests)

```
SendFundsFragment
  → SendFundsUiLogic.prepareTransactionForSigning()
    → ErgoFacade.prepareSerializedErgoTx()   // online reduce (appkit)
  → AbstractAuthenticationFragment → SigningSecrets
  → SendFundsUiLogic.startPaymentWithMnemonicAsync()
    → ErgoFacade.signSerializedErgoTx()      // ColdErgoClient
    → ErgoFacade.sendSignedErgoTx()          // RestApiErgoClient → node
```

ErgoPay: `ErgoPaySigningUiLogic` fetches request → `epsr.reducedTx` → same sign/send.

Cold: `ColdWalletUtils.buildColdSigningRequest` JSON + base64 reducedTx, QR-chunked (EIP-19).

Use these fixtures as S2 oracles. Do not port the Kotlin.
