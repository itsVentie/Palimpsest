# Palimpsest

Palimpsest is an open-source, local-first, zero-knowledge mobile privacy vault engineered for **iOS** and **Android**. Built on top of **Tauri v2**, **Preact**, and **Redb**, the application enables users to persistently store sensitive personal data—such as credentials, documents, metadata, and key-value records—strictly on-device with end-to-end cryptographic protection.

---

## Core Principles & Design Philosophy

* **Zero-Knowledge Architecture:** Data is encrypted prior to disk operations. No remote servers, central databases, key escrow, cloud relay, or telemetry exist.
* **Local-First Storage:** The user maintains absolute ownership of all stored artifacts. Data resides exclusively in application-isolated device storage.
* **Minimal Attack Surface:** Built using a low-overhead stack to minimize runtime memory footprint, eliminate heavy C-bindings, and shorten code execution paths during encryption/decryption routines.
* **Pure Rust Backend:** Avoids external C dependencies for database engines, enabling transparent cross-compilation across ARM targets and reducing exposure to legacy memory-safety vulnerabilities.

---

## Technical Stack Overview

| Tier | Technology | Purpose |
| --- | --- | --- |
| **User Interface** | Preact + Vite | Ultra-lightweight reactive Webview rendering (~4KB runtime bundle) for immediate cold-start response. |
| **Application Runtime** | Tauri v2 | Cross-platform Rust IPC bridge interfacing between the mobile Webview container and host operating system. |
| **Embedded Database** | Redb | Pure-Rust key-value engine supporting full ACID transactions and lock-free concurrenct read operations. |
| **Key Derivation** | Argon2id | Memory-hard password hashing function designed to resist GPU/ASIC brute-force side-channel attacks. |
| **Symmetric Encryption** | ChaCha20-Poly1305 | Authenticated Encryption with Associated Data (AEAD) optimized for mobile ARM architectures. |
| **Hardware Security** | iOS Keychain / Android Keystore | Native enclave storage for secure key caching paired with OS-level biometric prompts. |

---

## Security & Threat Model

### Key Generation and Management

1. **Passphrase Hashing:** The primary user passphrase passes through an Argon2id key derivation function using random salt generation. Parameters are configured to require significant execution memory and computation cycles to deter brute-force attempts.
2. **Ephemeral Memory Storage:** Key material in runtime memory is wrapped in strict zeroization containers that overwrite buffer memory directly when dropped or when the application transitions to the background.
3. **Biometric Integration:** Users can delegate session key unlock operations to hardware-backed secure enclaves (Apple Secure Enclave or Android KeyStore), protected via Face ID, Touch ID, or System Biometric authentication.

### Data at Rest Protection

* **Application Level AEAD:** Redb functions as a key-value store containing binary structures. Raw values undergo symmetric encryption using ChaCha20-Poly1305 before insertion.
* **Unique Nonce Per Record:** Every write operation generates a 96-bit cryptographically secure random nonce via hardware entropy sources. Nonces are prepended to the ciphertext and stored together.
* **Data Tamper Detection:** Poly1305 authentication tags validate payload integrity during read sequences. Any data mutation on disk causes complete decryption failure, preventing corrupted payload evaluation.

---

## System Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                        PREACT FRONTEND (WEBVIEW)                       │
│  - User Authentication Views                                           │
│  - Vault Record Management Interface                                   │
│  - State Management & In-Memory Render Buffers                         │
└──────────────────────────────────┬─────────────────────────────────────┘
                                   │  IPC Command Bridge
┌──────────────────────────────────▼─────────────────────────────────────┐
│                          TAURI V2 RUST BACKEND                         │
│                                                                        │
│   ┌───────────────────────┐                  ┌──────────────────────┐  │
│   │ Key Derivation Engine │                  │  Biometric Enclave   │  │
│   │      (Argon2id)       │                  │ (Keychain / Keystore)│  │
│   └───────────┬───────────┘                  └──────────┬───────────┘  │
│               │                                         │              │
│               └───────────────────┬─────────────────────┘              │
│                                   │ Master Key Material                │
│                                   ▼                                    │
│                     ┌────────────────────────────┐                     │
│                     │ Encryption Engine (AEAD)   │                     │
│                     │   (ChaCha20-Poly1305)      │                     │
│                     └─────────────┬──────────────┘                     │
│                                   │ Encrypted Binary Blobs             │
│                                   ▼                                    │
│                     ┌────────────────────────────┐                     │
│                     │  Redb Embedded KV Store    │                     │
│                     │    (On-Disk Encrypted)     │                     │
│                     └────────────────────────────┘                     │
└────────────────────────────────────────────────────────────────────────┘

```

---

## Build Prerequisites

### Base Requirements

* **Rust**: Toolchain release `1.75+` with ARM mobile targets (`aarch64-linux-android`, `aarch64-apple-ios`).
* **Node.js**: LTS Release `18+` or equivalent package manager (`pnpm`/`bun`).

### Mobile Build Environments

* **Android**: Android Studio with NDK `25+`, CMake, and configured SDK build-tools environment paths (`ANDROID_HOME`, `NDK_HOME`).
* **iOS**: macOS system with Xcode `15+` command line tools, CocoaPods, and valid Apple Developer target provisioning profiles.

---

# Roadmap

<details>
<summary><b>Phase 1: Environment Setup & Cryptographic Core</b></summary>

- [ ] **Development Environment Preparation**
  - [ ] Configure Rust toolchain (`1.75+`) and target architectures (`aarch64-linux-android`, `aarch64-apple-ios`).
  - [ ] Setup Android Studio (NDK 25+, CMake) and Xcode 15+ environments.
  - [ ] Initialize the Tauri v2 project structure with a Preact + Vite template.
- [ ] **Cryptographic Primitive Engine**
  - [ ] Implement master key derivation using `Argon2id` with memory-hard parameters.
  - [ ] Implement symmetric authenticated encryption (AEAD) routines using `ChaCha20-Poly1305`.
  - [ ] Write secure memory wrapper abstractions using the `zeroize` crate to ensure non-volatile key cleanup on process drop/suspension.
- [ ] **Unit Testing & Verification**
  - [ ] Create Rust unit tests for key derivation reproducibility and salt generation.
  - [ ] Test vector verification for encryption/decryption cycles and authentication tag enforcement.

</details>

<details>
<summary><b>Phase 2: Embedded Database Layer & Data Engine</b></summary>

- [ ] **Redb Storage Subsystem**
  - [ ] Integrate `redb` as the core key-value storage engine within the Tauri Rust backend.
  - [ ] Define core transaction handlers for read, write, update, and delete operations.
  - [ ] Implement table definitions for metadata, encrypted vault payloads, and application state.
- [ ] **Payload Serialization & Encryption Pipeline**
  - [ ] Build the binary serializer/deserializer to transform input JSON data into encrypted byte arrays.
  - [ ] Implement automatic 96-bit random nonce generation per write transaction.
  - [ ] Construct full data read pathways: Payload Fetch → Nonce Extraction → AEAD Decryption → Zeroization of temporary buffers.
- [ ] **Database Migration System**
  - [ ] Design schema versioning markers inside `redb` to allow smooth future upgrades.

</details>

<details>
<summary><b>Phase 3: Tauri IPC & Biometric Integration</b></summary>

- [ ] **Tauri v2 IPC Command Layer**
  - [ ] Implement Tauri commands for vault initialization, user authentication, and data manipulation.
  - [ ] Expose error handling interfaces to safely return error codes to the webview without leaking sensitive metadata.
- [ ] **Hardware Security & Biometrics**
  - [ ] Integrate `tauri-plugin-biometric` for Android (BiometricPrompt) and iOS (Face ID / Touch ID).
  - [ ] Integrate `tauri-plugin-stronghold` or native KeyStore / Keychain APIs to store and retrieve key derivatives securely.
- [ ] **App Lifecycle Management**
  - [ ] Attach listeners to OS lifecycle events (backgrounding, suspension, termination).
  - [ ] Implement instant key zeroization and state locking triggers when the application loses focus.

</details>

<details>
<summary><b>Phase 4: Preact UI Construction & State Management</b></summary>

- [ ] **Core Interface Views**
  - [ ] **Onboarding & Setup View:** Initial passphrase creation, entropy setup, and biometric enrollment.
  - [ ] **Unlock View:** Master passphrase entry and quick biometric trigger interface.
  - [ ] **Vault View:** Searchable record lists, category filters, and detailed payload view/edit modes.
  - [ ] **Settings View:** Security configuration, timeout settings, and manual vault locking.
- [ ] **State Management & Webview Security**
  - [ ] Build lightweight state containers in Preact using hooks.
  - [ ] Implement auto-blur overlay to hide sensitive UI data when switching app tasks in the OS multitasking view.
  - [ ] Ensure plaintext values in memory are wiped when views unmount.

</details>

<details>
<summary><b>Phase 5: Hardening, Backup Systems & Auditing</b></summary>

- [ ] **Encrypted Backup Engine**
  - [ ] Design an off-device backup spec (single encrypted file envelope derived via separate Argon2id parameters).
  - [ ] Implement backup export and import mechanisms via mobile native file pickers.
- [ ] **Hardening & Anti-Tampering**
  - [ ] Configure release profiles with Symbol Stripping (`strip = true`), Link-Time Optimization (`lto = true`), and Opt-Level 3.
  - [ ] Disable debug log outputs in production builds.
  - [ ] Enforce strict Content Security Policy (CSP) in the mobile webview.
- [ ] **Security & Performance Auditing**
  - [ ] Memory leak inspections and verification of zeroization routines under low-memory OS kills.
  - [ ] Performance measurements for database read/write throughput and UI cold-start time.

</details>

<details>
<summary><b>Phase 6: Mobile Compilation & Distribution</b></summary>

- [ ] **Android Target Release**
  - [ ] Configure Android manifest permissions (`USE_BIOMETRIC`).
  - [ ] Generate signed Android Application Bundles (AAB) and APKs.
  - [ ] Test on physical ARM64 devices across varying Android OS versions.
- [ ] **iOS Target Release**
  - [ ] Configure `Info.plist` key declarations (`NSFaceIDUsageDescription`).
  - [ ] Build via Xcode toolchain and generate iOS App Store packages (`.ipa`).
  - [ ] Verify background lifecycle behavior on physical iOS hardware.

</details>

---


## Operational Verification & Security Guidelines

* **Compiler Protections:** Production builds enforce release-profile optimizations, dead-code elimination, and full symbol stripping to resist binary reverse-engineering.
* **Prohibited Log Output:** Debug print streams are conditionally compiled out in production targets to prevent leakages of transient keys, memory pointers, or plaintext references.
* **Process Lifecycle Handling:** Backgrounding signals on mobile platforms trigger immediate zeroization routines for in-memory session keys, forcing re-authentication upon app resume.