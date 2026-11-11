# Building and Running REA from Source on Windows

This document details the exact, verified commands and configurations required to resolve the **Local Node.js Runtime Mismatch (`v24.12.0`)**, restore the gitignored **Windows Native Authority Addon (`rea-windows-x64.node`)**, and build and run the REA Model Context Protocol (MCP) server on Windows using PowerShell.

To keep `rea-readonly` untouched, perform all build, extraction, and runtime operations in a separate working copy (`D:\Projects\Skills\re-lab\rea-run`) or via the published npm package.

---

## 1. Hard Blocker 1: Local Node.js Runtime Mismatch (`v24.12.0`)

### Root Cause (Verified in Source)
- **Installed System Version**: Node.js `v24.12.0` fails REA's startup admission gate.
- **Permitted Engine Range**: `^22.23.4 || ^24.22.1 || >=26.12.1` (Source: `package.json:46`).
- **Pinned Development Version**: **Node.js 24.22.1** (Source: `.nvmrc:1`).
- **Admission Gate Mechanism**: Before importing any compiled module, `scripts/rea.mjs:22-29` calls `unsupportedRuntimeMessage(packageJson.engines?.node, process.versions.node)` in `scripts/lib/runtime-admission.mjs:14-54`. Because `24.12.0` has minor version `12 < 22` (`scripts/lib/runtime-admission.mjs:51`), REA writes the following refusal to stderr and exits with code `1`:
  ```text
  REA requires Node.js 22.x (>=22.23.4), 24.x (>=24.22.1), 26.12.1+; this process runs Node.js 24.12.0. Install a supported release, then retry.
  ```
- **Why You Must Not Bypass `runtime-admission.mjs`**: As documented in `scripts/lib/runtime-admission.mjs:4-7`, compiled REA modules import Node APIs absent from earlier point releases; removing the check causes module-linking crashes.

### How to Fix the Node.js Version on Windows
Install **Node.js >= 24.22.1** (recommended, matching `.nvmrc:1`) or **Node.js 22 LTS >= 22.23.4** using any standard Windows method:

1. **Option A: Fast Node Manager (`fnm`) or `nvm-windows` (Recommended — keeps other projects on 24.12.0)**
   ```powershell
   # Using fnm:
   fnm install 24.22.1
   fnm use 24.22.1
   node --version   # Must print v24.22.1 (or >=24.22.1 / >=22.23.4)
   ```
2. **Option B: Official Windows Installer / `winget`**
   ```powershell
   winget install OpenJS.NodeJS.LTS
   # Or download the Node.js >=24.22.1 or >=22.23.4 x64 .msi from https://nodejs.org/
   ```
3. **Pinning the Node Executable for MCP Clients**:
   Even if your default system `PATH` remains on `v24.12.0`, you can point your MCP client configuration directly to the absolute path of the supported `node.exe` (see Section 5 below).

- **Package Manager**: `npm@11.16.0` (Source: `package.json:6`).

---

## 2. Hard Blocker 2: Restoring the Gitignored Windows Native Addon (`rea-windows-x64.node`)

### Root Cause (Verified in Source)
- REA ships a prebuilt Windows x64 Node-API 8 addon (`native/windows/build/rea-windows-x64.node` and `native/windows/build/manifest.json`, `package.json:91-93`, `src/domain/windowsNativeArtifact.ts:8-14`).
- Users do **not** compile this addon with Visual Studio or Python (`native/windows/README.md:3-4`). In fact, `scripts/build-windows-native.mjs:44-47` cross-compiles it exclusively on Linux via MinGW-w64 (`x86_64-w64-mingw32-g++`).
- However, `native/windows/build/` is **ignored by Git** and omitted from a raw source checkout (`native/windows/README.md:24,31`).
- On Windows (`win32`), `PrivateRuntimeRoot.create()` (`src/process/PrivateRuntimeRoot.ts:58-68`) and `spawnOwnedProviderProcess()` (`src/process/ProviderProcess.ts:289-302`) require this addon (`src/windows/WindowsNativeLoader.ts:89-114`). Without it, worker-backed tools (`analyze_javascript_application`, `analyze_web_bundle`, `inspect_web_network_capture`, `trace_web_source_location`) fail on Windows with:
  ```text
  Missing packaged Windows native file native/windows/build/manifest.json (ENOENT). Restore both native/windows/build/manifest.json and native/windows/build/rea-windows-x64.node from the rea-agents@6.8.0 tarball or build them with native/windows/README.md; no native artifact was loaded.
  ```

### Official Restoration Procedure (`native/windows/README.md:31-48`)
Extract `manifest.json` and `rea-windows-x64.node` from the published `rea-agents@6.8.0` npm tarball (whose version matches `"version": "6.8.0"` in `package.json:3`) into your `rea-run` working copy, then verify its SHA-256 and PE headers with `scripts/verify-windows-native-artifact.mjs:1-42` (shown in Step 2 below).

---

## 3. PowerShell Commands to Clone, Restore Addon, Install, Build, and Start

Run the following commands in order in a standard PowerShell session using your supported Node.js (`>=24.22.1` or `>=22.23.4`):

### Step 1: Clone into a Separate Working Copy and Verify Node
```powershell
git clone D:\Projects\Skills\re-lab\rea-readonly D:\Projects\Skills\re-lab\rea-run
cd D:\Projects\Skills\re-lab\rea-run
git checkout dbfab4a65db9bba84b5cde652048401fb19c153d
node --version   # Must satisfy ^22.23.4 || ^24.22.1 || >=26.12.1 (package.json:46)
```

### Step 2: Restore the Windows Native Addon from `rea-agents@6.8.0`
- **Source**: `native/windows/README.md:31-48`, `src/windows/WindowsNativeLoader.ts:89-90`, `scripts/verify-windows-native-artifact.mjs:27-42`.
```powershell
npm pack rea-agents@6.8.0
tar -xzf rea-agents-6.8.0.tgz
New-Item -ItemType Directory -Force -Path native\windows\build | Out-Null
Copy-Item package\native\windows\build\manifest.json native\windows\build\
Copy-Item package\native\windows\build\rea-windows-x64.node native\windows\build\
Remove-Item -Recurse -Force package, rea-agents-6.8.0.tgz
```

### Step 3: Install Locked Dependencies
```powershell
npm ci
```
- **Source**: `README.md:120`, `docs/getting-started/installation.md:613`, `AGENTS.md:55`.
- **Purpose**: Installs the exact locked dependencies from `package-lock.json` without modifying the lockfile (`prepare` runs `scripts/prepare.mjs:1-39`).
- **Security Note**: `npm ci` runs dependency install scripts; review the scripts section of `package.json` first; whether the build works with `--ignore-scripts` is **UNVERIFIED**.

### Step 4: Verify the Windows Native Artifact and Build the Runtime
```powershell
node scripts/verify-windows-native-artifact.mjs
npm run build:cached
```
- **Source**: `package.json:309` (`verify:windows-native:artifact`), `docs/getting-started/installation.md:614`, `package.json:146`, `AGENTS.md:56`.
- **Node 26 Build Note (`scripts/lib/verifier-source-loader.mjs:98-102`)**: Node 26.x restricts `nodeModule.stripTypeScriptTypes` to `mode: "strip"` and rejects TypeScript constructor parameter properties (`src/domain/skillManifest.ts:15`). When building under Node 26.x (instead of `.nvmrc`'s `24.22.1`), add a `try/catch` around `nodeModule.stripTypeScriptTypes` in `scripts/lib/verifier-source-loader.mjs` that falls back to `createRequire(import.meta.url)("esbuild").transformSync(source, { loader: "ts", format: "esm", sourcefile: url, sourcemap: "inline" }).code`.
- **Underlying Build Script**: `node scripts/check-dependency-install.mjs && node scripts/run-exclusive.mjs artifacts node scripts/run-turbo.mjs run artifacts:skills` (Source: `package.json:146`).
- **Full Clean Rebuild (Optional)**: `npm run build` runs with `--force` (Source: `package.json:126`).

### Step 5: Start the MCP Server Standalone
```powershell
node scripts/rea.mjs mcp
```
or via npm script:
```powershell
npm start
```
- **Source**: `package.json:158` (`"start": "node scripts/rea.mjs mcp"`), `scripts/rea.mjs:46-79`, `docs/getting-started/installation.md:1934`.
- **Mechanism**: `scripts/rea.mjs` admits the Node version (`scripts/rea.mjs:22-29`), detects `args[0] === "mcp"`, sets `isMcpMode = true`, verifies compiled artifacts in `dist/mcp/main.js`, dynamically imports `runEntrypoint`, and listens for JSON-RPC messages over stdio.

---

## 4. How to Confirm the Server Starts on Windows

1. **Runtime & Native Authority Verification**:
   - `scripts/rea.mjs:72-76` verifies that `dist/mcp/main.js` and `dist/cli-runtime/cliBuiltinArguments.js` exist. If absent, it writes:
     `"REA's compiled runtime is missing. Run \`npm ci && npm run build:cached\`..."` and exits with code 1.
   - Run `node scripts/rea.mjs --version` (`scripts/rea.mjs:22-29`) to confirm Node runtime admission succeeds.
   - Run `node scripts/rea.mjs doctor` (`scripts/rea.mjs:48,83-90`) to confirm environment health and Windows native authority readiness (`src/process/WindowsAuthority.ts:57-87`).
2. **MCP JSON-RPC Handshake Confirmation**:
   - When launched with `node scripts/rea.mjs mcp`, REA listens silently on stdio for JSON-RPC 2.0 messages.
   - Sending an MCP initialization payload via stdin:
     ```json
     {"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test-client","version":"1.0.0"}}}
     ```
     produces a JSON-RPC response on stdout containing server capabilities and info (`scripts/rea.mjs:77-79`).
3. **Unverified Behavior**:
   - Whether running `node scripts/rea.mjs mcp` in an interactive terminal prints any human-visible banner without stdin input is **UNVERIFIED** (source inspection indicates it silently awaits JSON-RPC input on stdio).

---

## 5. Client Startup and MCP Configuration Snippet

MCP clients (such as Claude Desktop, Cursor, Codex, Gemini CLI, Antigravity, or custom harnesses) launch REA as a child process communicating over `stdio` (`docs/getting-started/installation.md:1931-1937`, `docs/getting-started/client-configuration.md:877-885`).

If your global `PATH` still points to Node `v24.12.0`, set `"command"` to the **absolute path** of your supported `node.exe` (e.g. `24.22.1`):

```json
{
  "mcpServers": {
    "rea": {
      "command": "C:\\Path\\To\\Supported\\Node24.22.1\\node.exe",
      "args": [
        "D:\\Projects\\Skills\\re-lab\\rea-run\\scripts\\rea.mjs",
        "mcp"
      ]
    }
  }
}
```

---

## 6. Windows Tool Support Matrix & Workarounds

| Tier | Tools | Windows Status (Source-Verified) | Windows Action / Workaround |
| :--- | :--- | :--- | :--- |
| **Tier 1: In-Process & CDP Connect** | `list_browser_targets`, `inspect_web_page`, `observe_web_session`, `compare_web_screenshots`, `export_web_scripts`, `trace_web_module_imports`, `trace_application_feature`, `trace_javascript_semantics`, `compare_application_versions`, `compare_static_and_observed_endpoints`, `inspect_electron_fuses`, `list_electron_targets`, `inspect_electron_page`, `inspect_analysis_view`, `verify_reconstruction`, `export_environment_bundle`, `export_evidence_bundle`, `reset_investigation` | No OS restriction in source; runs in Node.js process or via loopback CDP (`http://127.0.0.1:9222`) | Start Chrome/Edge manually with `--remote-debugging-port=9222 --user-data-dir="C:\Temp\cdp-profile"` for live CDP tools. |
| **Tier 2: Worker & PrivateRuntimeRoot Tools** | `analyze_javascript_application`, `analyze_web_bundle`, `inspect_web_network_capture` (`format: "har" \| "saz" \| "qlog"`), `trace_web_source_location` | Uses `OwnedWorkerRuntime` (`src/process/OwnedWorkerRuntime.ts:340`) or `PrivateRuntimeRoot` (`src/process/PrivateRuntimeRoot.ts:54`), which require `native/windows/build/rea-windows-x64.node` on local NTFS (`src/windows/WindowsNativeLoader.ts:89-114`). | Complete **Step 2** above to restore `manifest.json` and `rea-windows-x64.node` into `native\windows\build\`, and ensure `%TEMP%` is on local NTFS. |
| **Tier 3: Active Playwright Scenarios** | `capture_browser_scenario`, `capture_electron_scenario` | Launch boundary tests skipped on `win32` (`src/browser/PlaywrightScenarioBrowser.test.ts:99`; `docs/specs/agent-environments/scenario-capture/verification.md:139`). | For `capture_browser_scenario`, prefer **connect mode** (`browser: { mode: "connect", cdp_endpoint: "http://127.0.0.1:9222" }`) against an already-running Chrome instance, or supply `browser: { mode: "launch", executable_path: "C:\\Program Files\\Google\\Chrome\\Application\\chrome.exe" }`. |
| **Tier 4: Linux-Only Tools** | `recover_javascript_sources`, `inspect_web_network_capture` (`format: "mitmproxy"`) | Hard OS gate: requires Linux x64 and `/usr/bin/prlimit` (`src/javascript/recovery/WakaruCommand.ts:100`) or Linux `mitmdump` (`src/contracts/webNetworkCaptureToolContracts.ts:19`). | Do not call `recover_javascript_sources` natively on Windows; run external `webcrack`/`wakaru`/`prettier` (under Node.js or WSL2) and pass the unpacked folder to `analyze_javascript_application` (`input_path`). Use `.har` or `.saz` instead of `mitmproxy`. |

---

## 7. Explicit Unknowns and Unverified Items

- **UNVERIFIED**: Whether the build works with `npm ci --ignore-scripts` (dependency install scripts run by default; review `package.json` scripts first).
- **UNVERIFIED**: Whether Windows Smart App Control or App Control for Business policies on a given host block loading unsigned or third-party `.node` binaries (`src/windows/WindowsNativeLoader.ts:93-95`).
- **UNVERIFIED**: Runtime execution of the Tier 1–3 tools on this specific Windows machine prior to upgrading Node.js from `v24.12.0` to `>=24.22.1` (or `>=22.23.4`).
