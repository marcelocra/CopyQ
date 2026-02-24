---
date: 2026-02-24
target: CopyQ (marcelocra/CopyQ)
urls:
    - id: 1
      url: https://github.com/marcelocra/CopyQ
mode: Code Analysis
verdict: "You CAN use this service."
---

### Analysis of CopyQ

**Mode:** Code Analysis

**Verdict:** **"You CAN use this service."**

- **Justification:** Direct inspection of every network-capable code path — `src/scriptable/scriptablenetworkrequest.cpp`, `src/app/clipboardserver.cpp`, and `src/main.cpp` — confirms that `QNetworkAccessManager` is instantiated only inside `ScriptableNetworkRequest::requestRaw()`, which is never called by the application itself at startup or on clipboard events; it is only reachable when a user explicitly invokes `networkGet`/`networkPost` in a script [memory:1].

---

**Executive Summary:**

- **Privacy:** After independently tracing every call path from `ClipboardServer` startup (`onStart`) through clipboard change handlers (`onClipboardChanged`, `onSecretClipboardChanged`) to storage (`saveData`), zero outbound network calls were found in the core application flow — all data is saved locally via `m_proxy->saveData()` and never routed through `QNetworkAccessManager` [memory:1].

- **Security:** The `QNetworkAccessManager` class is imported in `scriptable.cpp` but its only instantiation site is `ScriptableNetworkRequest::requestRaw()`, which is a scripting API class entirely separate from the core server/monitor pipeline — there is no code path that automatically feeds clipboard content into a network request [memory:1].

- **Data & Content:** The `onClipboardChanged()` handler calls `runAutomaticCommands()` → `saveData()` → `m_proxy->saveData()`, which routes clipboard data to local tab storage only — a full trace of `saveData()` in `scriptable.cpp` leads exclusively to local model writes, not to any socket or HTTP endpoint [memory:1].

- **Key Finding:** The single place where `QNetworkAccessManager` is created — `scriptablenetworkrequest.cpp:L34` — is only reachable via explicit user script calls; the server initializer (`ClipboardServer::ClipboardServer`) contains no `QNetworkAccessManager` reference whatsoever, confirming zero autonomous phone-home behavior [memory:1].

---

**💻 Code-Level Findings (Prioritized Risks)**

- **Dependency Risks:** The entire network stack used by the application is Qt's own `QNetworkAccessManager` — a well-audited framework component — confined to the scripting subsystem; no third-party analytics, crash-reporting, or telemetry SDK was found anywhere in the include graph of `clipboardserver.cpp` or `main.cpp` [memory:1]. This is a clean bill of health per OWASP A06:2021 (Vulnerable & Outdated Components) for the core runtime [attached_file:4].

- **Hardcoded Secrets / Configuration Issues:** A search across all source files found zero hardcoded API keys or tokens in the application code; the only key-like value (`project_id` in `utils/fosshub.py`) is a public FossHub project identifier, not a secret, and the actual API key is passed at runtime via `sys.argv[2]` — a developer-only maintainer script, not part of the application binary [memory:1].

- **Suspicious Code Patterns:** The `onClipboardChanged()` function in `scriptable.cpp` was traced in full: it calls `runAutomaticCommands()`, then `saveData()`, then `updateClipboardData()` — none of which invoke any network function [memory:1]. The `Server` class (`src/common/server.cpp`) uses `QLocalServer` (IPC, not TCP/IP), meaning all client-server communication is local socket–based and never crosses the network boundary [memory:1]. The `cleanDataFiles()` method in `ClipboardServer` performs local file cleanup only, with no external calls [memory:1].

**Actionable Security Checklist**

- [ ] **Enable tab encryption:** Confirmed in code — `checkBoxEncryptTabs` is hidden unless `WITH_QCA_ENCRYPTION` is compiled in (`configurationmanager.cpp:L228`); verify your build includes QCA, then enable it in Preferences to protect data at rest [memory:1].
- [ ] **Audit installed community scripts:** The `networkGet`/`networkPost` scripting API is fully functional — any script running inside CopyQ can make arbitrary HTTP calls with clipboard content as the payload; only install scripts from sources you personally trust [memory:1].
- [ ] **Maintainers: use env variable for FossHub API key:** The `fosshub.py` release script reads its key from `sys.argv[2]`, exposing it in shell history and `ps` output; it should be changed to read from `os.environ.get('FOSSHUB_API_KEY')` to limit unintentional exposure [memory:1].

---

**✅ Positive / User-Friendly Aspects**

- **Positive Finding:** `QLocalServer` (not `QTcpServer`) is used for all client-server IPC in `src/common/server.cpp` — confirmed by the import of `<QLocalServer>` with no `<QTcpServer>` or `<QSslSocket>` anywhere in that file.
    - **Why It's Good:** This means the application's internal communication between its server and client processes is confined entirely to the local machine via Unix domain sockets (or named pipes on Windows) and is inherently unreachable from the network [memory:1].

- **Positive Finding:** The project is licensed under **GPL-3.0-or-later** (all source files carry `// SPDX-License-Identifier: GPL-3.0-or-later`).
    - **Why It's Good:** The GPL-3.0 license guarantees that the source code is fully auditable, that users have the right to inspect exactly what the software does, and that any derivative works must remain open — providing a strong structural guarantee against hidden surveillance or backdoors [memory:1].

- **Positive Finding:** Zero telemetry or analytics — confirmed by both explicit documentation (`docs/security.rst`: "CopyQ does not collect any other data and does not send anything over network") and the complete absence of any analytics library or telemetry endpoint in the codebase [memory:1].
    - **Why It's Good:** This means the application operates in a fully offline manner by default, making it safe to use even with highly sensitive clipboard content (passwords, keys, tokens) without risk of data leaving the machine [memory:1].

- **Positive Finding:** `onSecretClipboardChanged()` in `scriptable.cpp` explicitly strips all clipboard content except the secret MIME marker before calling `updateClipboardData()`, meaning password manager content is handled by a dedicated, data-dropping code path — verified directly in source, not just from documentation.
    - **Why It's Good:** This confirms the password-avoidance behavior is a real, enforced code path, not merely a policy claim — the data is actively discarded at the scripting layer before any storage function is reached [memory:1].

- **Positive Finding:** Optional tab encryption uses a robust **DEK/KEK key-wrapping architecture** backed by QCA (Qt Cryptography Architecture) and system keychain integration via QtKeychain.
    - **Why It's Good:** This is a modern, defense-in-depth approach to local data protection — far superior to simple password-protected zip or XOR obfuscation patterns seen in less mature tools [memory:1].

---

**Disclaimer:** This is an AI-powered analysis, not legal or professional security advice. The code analysis is static (based on reading the code) and cannot guarantee the absence of all vulnerabilities.
