# Notice

This derivative work (`reverse-engineer-web`) is derived from the REA ("Reverse Engineer Anything") repository by morluto (commit `dbfab4a65db9bba84b5cde652048401fb19c153d`, version 6.8.0), licensed under the MIT License (Copyright (c) 2026 morluto). It restructures the original agent skill into a compact, web-first reverse engineering workflow designed to preserve critical rules, guardrails, and authorization gates under agent context truncation, moves non-web routing into modular reference files, incorporates documented contracts for web/browser/runtime tools (`analyze_web_bundle`, `observe_web_session`, `capture_electron_scenario`, `compare_web_screenshots`, `inspect_electron_page`, `list_javascript_runtime_targets`, `observe_javascript_runtime`, `trace_javascript_semantics`, `trace_web_module_imports`, `trace_web_source_location`), adds dedicated clean-room specification and differential verification guides, removes external build-computer dependencies, and documents local execution workflows.

---

## Version Gap (npm rea-agents@6.8.0 vs. Source Commit dbfab4a)

| Dimension / Feature | Published npm `rea-agents@6.8.0` (2026-10-25) | Repo Source Commit `dbfab4a` (Head) | Skill Impact & Fallback |
| :--- | :--- | :--- | :--- |
| **Release & Bundled Skill** | Published 2026-10-25, bundled skill v49 (506 lines) | Commit dbfab4a, bundled skill v62 | Skill targets dbfab4a contracts |
| **`reset_investigation`** | Lacking (not present in package) | Added in commit `f72638f6b` (2026-11-01) | If missing, skip call; do not substitute guesses |
| **`save_path` parameter** | Lacking | Added to `reset_investigation` schema (`f72638f6b`) | Checkpoint parameter only on source builds |
| **Evidence retention across cleanup** | Evidence cleared on `close_binary` | Retention across cleanup changed (`f72638f6b`) | On npm 6.8.0, evidence may not survive `close_binary` |
| **`compare_static_and_observed_endpoints`** | Lacking (not present in package) | Added in commit `3708538e8` (2026-11-06) | If missing, compare manually via HAR |
| **`capture_browser_scenario` `detail`** | Only default detail | Added `detail` parameter (`fc43e285e`) | (source build only) |
| **`trace_application_feature` `semantic_page`** | Lacking `semantic_page` | Added `semantic_page` parameter (`8eff65127`) | (source build only) |
| **`inspect_web_network_capture` budgets** | Fixed budgets only | Configurable `max_input_bytes`, `max_reply_bytes`, `max_nesting_depth` (`cb4e5cfbf`) | (source build only) Defaults apply if omitted |
| **`trace_web_source_location` `detail` & `views`** | Basic position lookup only | Added `detail` and `views` parameters (`cb4e5cfbf`) | (source build only) Base inputs work on both |
