# Instructions for AI Coding Assistants (AGENT.md)

Welcome, fellow AI Assistant! This repository contains `schwabdev`, a lightweight Python wrapper for the Charles Schwab Developer API, and a built-in local MCP (Model Context Protocol) server. 

When modifying, extending, or using this codebase, please follow the guidelines, architectural patterns, and safety constraints outlined below.

---

## 1. Safety & The Order Safety Protocol

Charles Schwab API access allows real trade executions. To prevent accidental trade execution or incorrect quantity entry, you **must** follow the **Order Safety Protocol** when writing code or assisting the user:

1. **Never Construct Ad-hoc Payloads**: Always use the predefined builder tools (`build_equity_market_order`, `build_equity_limit_order`, `build_option_order`) to generate standard Schwab order payloads.
2. **Mandatory Dry-run Previews**: Before executing any trade (`place_order`), you **must** pass the order payload to `preview_order` first.
3. **Explicit Consent**: You **must** present the estimated costs, commissions, and buying power impact from the preview to the user and await their explicit confirmation before calling `place_order`.

---

## 2. Core Repository Architecture

### A. Dual Client Architecture
Schwabdev supports both synchronous and asynchronous modes of operation. Maintain this separation strictly:
* **Synchronous Client (`schwabdev.Client`)**: Standard thread-blocking client. Best for sequential scripts, simple automations, and standard MCP tool handlers.
* **Asynchronous Client (`schwabdev.ClientAsync`)**: Uses `aiohttp` for non-blocking event loops. Best for high-frequency streaming applications or asynchronous background loops.
* **WebSocket Streamers (`Stream` and `StreamAsync`)**: Used for real-time market data subscriptions.

### B. SQLite Tokens Database
Schwabdev stores OAuth tokens in a local SQLite database (`tokens.db`), allowing multiple clients (sync/async) to share and read tokens concurrently without conflict.
* The database path is configurable (defaults to `~/.schwabdev/tokens.db`).
* Support for database encryption is optional using `cryptography.fernet`. Refer to `docs/examples/extra/encrypted_db_setup.py`.

### C. Headless / Non-Interactive Authentication
Schwab API refresh tokens expire every 7 days. 
* To prevent processes from deadlocking in headless/stdio environments (such as within an MCP server), Schwabdev allows passing a custom `call_on_auth` handler to `Client`.
* In `mcp_server.py`, the callback raises `AuthenticationRequiredError` containing instructions and a browser authorization link, letting the calling assistant direct the user on how to re-authenticate cleanly.

### D. Documentation System
The documentation site is hosted under `docs/`.
* The documentation pages are generated using a simple markdown compiler: [compile-docs.py](docs/compile-docs.py).
* **Never edit files in `docs/pages/` directly.** Instead, make your edits to the raw Markdown files in [docs/pages-raw/](docs/pages-raw/) and then execute the compiler to generate the HTML pages:
  ```bash
  python docs/compile-docs.py
  ```

---

## 3. General Implementation Rules

* **Preserve Documentation**: Do not remove, modify, or truncate existing Docstrings or comments in `schwabdev` unless explicitly requested.
* **Lazy Client Instantiation**: If you write tools or plugins (like the MCP server), lazily initialize the `Client` on the first tool invocation. This prevents startup crashes due to missing environment configurations.
* **Type passthrough**: Date parameters should be passed through as ISO 8601 strings, which the core client forwards directly to the REST endpoints.
* **Response Truncation**: When querying endpoints that return massive lists (e.g. option chains), always implement truncation boundaries (~50,000 characters) to prevent context-window blowups for LLM clients.
