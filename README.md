# rea-web-skill: Web-First Reverse Engineering for Antigravity

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Platform: Windows / Linux / macOS](https://img.shields.io/badge/Platform-Windows%20%7C%20Linux%20%7C%20macOS-brightgreen.svg)](#windows-host-compatibility)
[![REA Engine: 6.8.0](https://img.shields.io/badge/REA%20Engine-6.8.0-blueviolet.svg)](https://github.com/morluto/rea)

A specialized, web-first reverse engineering skill for [Google Antigravity](https://deepmind.google/technologies/antigravity/) and AI coding agents. Derived from [morluto/rea](https://github.com/morluto/rea) at commit `dbfab4a` (v6.8.0, MIT License).

`rea-web-skill` streamlines the analysis of frontend JavaScript bundles, minified scripts, web APIs, network captures (HAR/SAZ), and Electron desktop applications into clean-room architectural specifications and verified code reconstructions.

---

## Repository Structure

```text
rea-web-skill/
├── reverse-engineer-web/                  # The Antigravity skill package
│   ├── SKILL.md                          # Skill entry point and 6-phase routing workflow
│   ├── LICENSE                           # MIT license
│   ├── NOTICE.md                         # Upstream copyright and attribution notice
│   └── references/                       # 11 focused reference guides
│       ├── web-reverse-engineering.md    # Primary web & bundle reverse engineering guide
│       ├── reconstruction-and-verification.md # Clean-room clone specs & verification ledgers
│       ├── authorization-and-clean-room.md   # Ethical boundaries and clean-room provenance
│       ├── connection-and-recovery.md    # MCP server recovery, diagnostics, and repairs
│       ├── javascript-applications.md    # AST analysis and framework-specific patterns
│       ├── runtime-observation.md        # CDP browser attachment & scenario capture
│       ├── evidence-workflows.md         # Evidence lifecycle, exports, and hashing
│       ├── environments.md               # AgentEnv browser automation and bundles
│       ├── routing-native-and-artifacts.md # Hand-off routing for binary/firmware targets
│       ├── native-and-artifacts.md       # Native boundaries and Electron core analysis
│       └── android-applications.md       # Android/JADX hand-off reference
├── RUN_FROM_SOURCE.md                    # Windows source build guide & Node 26 compatibility
├── CHANGES.md                            # Comprehensive derivation log, audit tables, and token budgets
├── NOTICE.md                             # Repository attribution notice
└── README.md                             # This file
```

---

## Key Features

1. **Strict Evidence Provenance**: Distinguishes observations, derivations, and inferences. Never substitutes guesses for unobserved facts.
2. **Deterministic Token Budgets**: `SKILL.md` is strictly capped at `≤ 3,000` estimated tokens (`2,987.75` tokens) for fast agent loading; references are capped at `≤ 6,000` tokens.
3. **Clean-Room Reconstruction**: Outlines 3-layer clone specifications (Design Tokens, Component Hierarchy, Behavioral Contracts) and reconstruction obligation ledgers.
4. **Verified Windows Native Support**: Documents exact workarounds for Node.js 26 LTS (`v26.13.0`), restoring prebuilt Windows native addons (`rea-windows-x64.node`), and running under Windows NTFS.
5. **Fail-Closed Authorization**: Strictly gates active network observation and browser interactions behind explicit user permission.

---

## Installation

### 1. Install the Skill into Antigravity

Copy the `reverse-engineer-web` folder into your global Antigravity skills directory:

**Windows (PowerShell):**
```powershell
Copy-Item -Path ".\reverse-engineer-web" `
  -Destination "$env:USERPROFILE\.gemini\config\skills\reverse-engineer-web" -Recurse -Force
```

**macOS / Linux:**
```bash
cp -r ./reverse-engineer-web ~/.gemini/config/skills/reverse-engineer-web
```

Or place it in a local project repository under `.agents/skills/reverse-engineer-web/`.

### 2. Configure the REA MCP Server

Add the `rea` MCP server to your Antigravity MCP configuration (`~/.gemini/antigravity/mcp_config.json`):

```json
{
  "mcpServers": {
    "rea": {
      "command": "node",
      "args": [
        "D:\\Projects\\Skills\\re-lab\\rea-run\\scripts\\rea.mjs",
        "mcp"
      ]
    }
  }
}
```

*(Adjust the path to match your local REA build directory. If running without the MCP server, the skill automatically uses the CLI fallback `node scripts/rea.mjs <command>`)*.

---

## Windows Host Compatibility

| Tier | Tools | Windows Support | Recommended Action / Workaround |
| :--- | :--- | :--- | :--- |
| **Tier 1: In-Process & CDP Connect** | `list_browser_targets`, `inspect_web_page`, `observe_web_session`, `compare_web_screenshots`, `export_web_scripts`, `trace_web_module_imports`, `trace_application_feature`, `trace_javascript_semantics`, `compare_application_versions`, `compare_static_and_observed_endpoints`, `inspect_electron_fuses`, `list_electron_targets`, `inspect_electron_page`, `inspect_analysis_view`, `verify_reconstruction`, `export_environment_bundle`, `export_evidence_bundle`, `reset_investigation` | **Supported** (No OS restrictions) | Start Chrome or Edge with remote debugging enabled:<br>`chrome.exe --remote-debugging-port=9222 --user-data-dir="%TEMP%\rea-chrome-profile"` |
| **Tier 2: Worker & PrivateRuntimeRoot** | `analyze_javascript_application`, `analyze_web_bundle`, `inspect_web_network_capture` (`format: "har" \| "saz" \| "qlog"`), `trace_web_source_location` | **Supported** (requires native addon) | Follow [RUN_FROM_SOURCE.md](RUN_FROM_SOURCE.md) to restore `manifest.json` and `rea-windows-x64.node` into `native\windows\build\`. Ensure `%TEMP%` is on local NTFS. |
| **Tier 3: Active Scenario Launch** | `capture_browser_scenario`, `capture_electron_scenario` | **Partial** (launch boundary skipped on Win32) | Use **connect mode** against an already running browser instance (`browser: { mode: "connect", cdp_endpoint: "http://127.0.0.1:9222" }`). |
| **Tier 4: Linux-Gated Tools** | `recover_javascript_sources`, `inspect_web_network_capture` (`format: "mitmproxy"`) | **Unsupported on Win32** (requires Linux `/usr/bin/prlimit`) | Run external deobfuscators (`webcrack`, `wakaru`, `prettier`) via Node.js or WSL2, then pass the output directory to `analyze_javascript_application`. |

For complete instructions on building REA from source on Windows with Node.js 26 LTS, see [RUN_FROM_SOURCE.md](RUN_FROM_SOURCE.md).

---

## Analysis Workflow

The skill executes through 6 structured phases:

```text
Phase 1: Intake & Scope Check
  └── Determine target type (Bundle, HAR, Live App, Electron) & confirm user authorization.
Phase 2: Static Reconnaissance
  └── Analyze bundles (analyze_web_bundle) & parse network captures (inspect_web_network_capture).
Phase 3: Runtime Observation (Opt-in)
  └── Attach via CDP (list_browser_targets, inspect_web_page) to extract live APIs and DOM.
Phase 4: Semantic Analysis & Feature Tracing
  └── Map modules (trace_web_module_imports), decode semantics (trace_javascript_semantics), and extract source locations.
Phase 5: Version & Endpoint Comparison
  └── Diff application bundles (compare_application_versions) and reconcile static vs live endpoints.
Phase 6: Specification & Verification
  └── Generate CLONE_SPEC.md, build reconstruction obligation ledgers, and run verify_reconstruction.
```

---

## Attribution & Provenance

This project is derived from the open-source [REA (Reverse Engineer Anything)](https://github.com/morluto/rea) framework:
- **Upstream Repository**: `https://github.com/morluto/rea`
- **Upstream Commit**: `dbfab4a` (October 25, 2026)
- **Upstream Author**: `morluto`
- **Original License**: MIT License (Copyright © 2026 morluto)

Full details of changes, rule verifications, and parameter schemas are documented in [CHANGES.md](CHANGES.md) and [NOTICE.md](NOTICE.md).

---

## License

This repository is distributed under the [MIT License](reverse-engineer-web/LICENSE).
