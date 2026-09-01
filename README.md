# Figma ↔ Claude Code

React + Vite + Tailwind project wired up so Claude Code can read from and write to a live Figma file — inspect designs, take screenshots, and create/edit nodes directly on the canvas.

## How the connection works

Three pieces, all running locally:

```
Figma Desktop app                    Claude Code (this terminal / IDE)
┌────────────────────────┐           ┌────────────────────────────┐
│  Your file open         │           │  figma-console MCP server   │
│  "Figma Desktop Bridge" │◄─ WS ────►│  (npx figma-console-mcp)    │
│  plugin running         │  ws://    │  registered as an MCP tool  │
│  (code.js + ui.html)    │  localhost│  provider for Claude        │
│                          │  :9223   │                              │
└────────────────────────┘           └────────────────────────────┘
```

1. **Figma Desktop Bridge** — a Figma plugin (manifest.json + code.js + ui.html) that runs *inside* Figma Desktop. When you click "Run" on it, it opens a WebSocket server on `localhost:9223` (falls back to 9224–9232 if busy).
2. **figma-console-mcp** — an MCP (Model Context Protocol) server that Claude Code launches on demand via `npx`. It connects to that same WebSocket port and exposes tools like `figma_execute`, `figma_take_screenshot`, `figma_get_status`, etc. to Claude.
3. **Claude Code** — has `figma-console` registered as an MCP server (global, user-level config). When you ask it to do something in Figma, it calls those MCP tools, which talk over the WebSocket to the plugin, which uses Figma's real Plugin API to read/write the actual file.

Nothing goes through Figma's REST API or cloud for the live editing — it's a direct local WebSocket bridge to the Desktop app. This is why **Figma Desktop (not the browser)** must be open with the file loaded and the plugin running.

## What's connected right now

- Figma file: **My workflow** — `https://www.figma.com/design/m9sLka8FFiMTvunBP31EIa/My-workflow`
- File key: `m9sLka8FFiMTvunBP31EIa`
- Page being edited: `Portfolio website`
- Bridge plugin source (cloned for reference/re-import): [`figma-console-mcp/figma-desktop-bridge/`](./figma-console-mcp/figma-desktop-bridge) (from https://github.com/southleft/figma-console-mcp)

## macOS setup — what was done (reference)

1. Installed/confirmed Node.js and npm (`node -v`, `npm -v`).
2. Registered the MCP server with Claude Code (user scope, so it's available in every project):
   ```bash
   claude mcp add figma-console -- npx -y figma-console-mcp@latest
   ```
   with environment variables set on that server entry:
   - `FIGMA_ACCESS_TOKEN` — a Figma personal access token (Figma → Settings → Security → Personal access tokens)
   - `ENABLE_MCP_APPS=true`

   Verify anytime with:
   ```bash
   claude mcp list
   ```
   `figma-console` should show `✔ Connected`.

3. Cloned the bridge plugin source for reference/import:
   ```bash
   git clone https://github.com/southleft/figma-console-mcp.git
   ```
4. In **Figma Desktop**: `Plugins → Development → Import plugin from manifest…` → picked `figma-desktop-bridge/manifest.json` from the cloned repo.

   (Note: the `figma-console-mcp` npm package also auto-installs its own copy of this same plugin at `~/.figma-console-mcp/plugin/manifest.json` the first time it runs — either manifest works, they're the same plugin.)
5. Opened the target file in Figma Desktop, ran the plugin: `Plugins → Development → Figma Desktop Bridge` → "Run". The plugin's floating panel shows **Connected — AI …** once the WebSocket handshake succeeds.
6. Confirmed from the terminal that the bridge's local server is listening:
   ```bash
   lsof -i :9223 -P -n | grep LISTEN
   ```
7. Confirmed from Claude Code that the MCP tool can actually reach Figma (asked Claude to check status / probe the connection) — should report `"✅ Connected to Figma Desktop via WebSocket Bridge"`.

## Daily use

1. Open Figma Desktop with the file.
2. Run the **Figma Desktop Bridge** plugin (`Plugins → Development → Figma Desktop Bridge → Run`). Leave its panel open — closing it drops the connection.
3. Open Claude Code in this project folder and just ask for what you need ("take a screenshot of the current page", "add a footer section", etc.). Claude calls the MCP tools itself.
4. To sanity-check the link from a plain terminal (no Claude tool call needed), check the port is listening:
   - **macOS/Linux:** `lsof -i :9223 -P -n | grep LISTEN`
   - This only proves the local server is up — it does **not** prove Figma's plugin is attached. Ask Claude to run a status probe for that.

## Frontend app (this repo)

Standard Vite + React 19 + Tailwind 4 scaffold.

```bash
npm install
npm run dev      # start dev server
npm run build     # production build
npm run preview   # preview the production build
npm run lint       # oxlint
```

---

## Windows setup — step by step (source of truth for the Windows machine)

> **Editor note:** the macOS setup above uses **Claude Code** (a terminal-based CLI). On Windows, this project uses **[Antigravity](https://antigravity.google/)** (Google's agentic IDE) instead — same MCP server (`figma-console`) and same Figma Desktop Bridge plugin, just a different client/editor wired up to them. The underlying bridge (Figma Desktop plugin ↔ WebSocket ↔ MCP server) is identical either way; only "how the MCP server gets registered" differs, covered in step 8 below.

Goal: reproduce the exact same setup as macOS, with zero version drift. Do these in order.

### 0. Versions to match (from the macOS machine)

| Tool | macOS version installed | What to install on Windows |
|---|---|---|
| Node.js | v26.7.0 | Node.js **v26.x** (same major line) — `.nvmrc` in this repo pins `26.7.0` exactly |
| npm | 11.19.0 | comes bundled with Node 26.x — no separate install |
| Client / editor | Claude Code 2.1.183 (CLI, native installer) | **Antigravity** (Windows installer — no Claude Code access on this machine) |
| Figma | Desktop app | Figma Desktop app for Windows |
| figma-console-mcp | pinned to **`1.40.0`** (not `@latest` — see note below) | same exact version, `1.40.0` |

> **Why pin `1.40.0` instead of `@latest`**: `@latest` resolves to whatever's newest at install time. Since macOS was set up first, using `@latest` again on Windows could silently pull a newer MCP server build with different tool behavior. Pinning to the exact version macOS is running on removes that risk entirely.

> Use **PowerShell** for everything below (not WSL/Git Bash) unless a step says otherwise. Reason: Figma Desktop for Windows is a native Windows app — if Node/Claude Code run inside WSL, the plugin manifest path and the loopback WebSocket can behave inconsistently across the WSL↔Windows network boundary. Keep everything on the native Windows side to match what worked on macOS.

### 1. Install Node.js

1. Download the Node.js **v26.x** Windows installer (`.msi`) from https://nodejs.org (choose the version that matches the major line above, not necessarily the exact patch).
2. Run the installer, keep defaults (this also installs npm and adds both to PATH).
3. Open a **new** PowerShell window and verify:
   ```powershell
   node -v
   npm -v
   ```
   Expect `v26.x.x` and `11.x.x` respectively. If npm's major version differs a lot from `11`, run `npm install -g npm@11` to align.

### 2. Install Git for Windows

1. Download from https://git-scm.com/download/win, install with defaults.
2. Verify: `git --version`

### 3. Install Antigravity

1. Download the Windows installer from https://antigravity.google/ and run it.
2. Launch Antigravity, sign in with your Google account when prompted.
3. This replaces Claude Code as the AI client for this project on Windows — the MCP server and Figma plugin underneath are the same, only the editor talking to them differs.

### 4. Install Figma Desktop for Windows

1. Download from https://www.figma.com/downloads/ → Figma Desktop for Windows.
2. Install, sign in, and open the **My workflow** file: https://www.figma.com/design/m9sLka8FFiMTvunBP31EIa/My-workflow

### 5. Get the project code onto the Windows machine

This repo currently has **no git remote configured** (it was only initialized locally on macOS). Before moving to Windows, push it somewhere you can clone from:

On macOS (one-time):
```bash
git remote add origin <your-repo-url>   # e.g. a GitHub repo you create
git push -u origin master
```

On Windows, then:
```powershell
git clone <your-repo-url> Figma_Claudecode
cd Figma_Claudecode
```

If you'd rather not push to a remote, copy the whole project folder over (external drive / cloud sync), **excluding** `node_modules` and `dist` (they're OS/arch-specific and will be reinstalled/rebuilt anyway — see `.gitignore`).

### 6. Install project dependencies

```powershell
cd Figma_Claudecode
npm install
```
This reads the same `package-lock.json` committed on macOS, so you get **identical dependency versions** (React 19.2.8, Vite 8.2.2, Tailwind 4.3.3, etc.) — no drift. The repo's `package.json` also pins `engines.node`/`engines.npm` and there's an `.nvmrc` (`26.7.0`) — if you use `nvm-windows`, run `nvm use` in the project folder to match exactly.

### 7. Get a Figma personal access token

1. In Figma (web or desktop): **Settings → Security → Personal access tokens → Generate new token**.
2. Copy it immediately (shown once). Do **not** reuse the macOS token if you'd rather keep them separate/revocable independently — either works, but generate your own to avoid having one token shared across two machines.

### 8. Register the figma-console MCP server with Antigravity

Antigravity reads MCP servers from a JSON config file, not a CLI command like Claude Code does.

1. Open the config file directly, or go through the UI: **Antigravity → Settings → Customizations tab → Open MCP Config**. Either way it points at:
   ```
   C:\Users\<you>\.gemini\antigravity\mcp_config.json
   ```
   (create the file/folders if they don't exist yet).

2. Set its contents to:
   ```json
   {
     "mcpServers": {
       "figma-console": {
         "command": "npx",
         "args": ["-y", "figma-console-mcp@1.40.0"],
         "env": {
           "FIGMA_ACCESS_TOKEN": "${FIGMA_ACCESS_TOKEN}",
           "ENABLE_MCP_APPS": "true"
         }
       }
     }
   }
   ```
   Note the pinned `figma-console-mcp@1.40.0` (see the version-pinning note above) instead of `@latest`.

3. `${FIGMA_ACCESS_TOKEN}` above is an environment-variable reference, not the literal token — **don't paste your token straight into this file**, since it's easy to accidentally commit or share. Instead set it as a real Windows environment variable once, in PowerShell (run as your user, not admin):
   ```powershell
   [System.Environment]::SetEnvironmentVariable("FIGMA_ACCESS_TOKEN", "<your-windows-token>", "User")
   ```
   Close and reopen Antigravity (and any terminal) afterward so it picks up the new env var.

4. Save the config file and restart Antigravity (or use its "reload MCP servers" action if the Customizations tab has one) so it picks up `figma-console`.

> **No conflict with macOS**: this config lives entirely in your Windows user profile (`~/.gemini/antigravity/mcp_config.json`), separate from Claude Code's own config store on macOS. Each machine keeps its own independent MCP registration and its own token — this file is **not** part of the git repo and should never be committed.

### 9. Import the Figma Desktop Bridge plugin

The cloned repo already contains the plugin at:
```
figma-console-mcp\figma-desktop-bridge\manifest.json
```
(present after step 5/6, since it's inside the project you cloned — if you're copying the folder manually instead of cloning fresh, make sure `figma-console-mcp/` came along too, or re-clone it separately: `git clone https://github.com/southleft/figma-console-mcp.git`)

In Figma Desktop:
1. `Plugins → Development → Import plugin from manifest…`
2. Browse to the Windows path, e.g.:
   ```
   C:\Users\<you>\...\Figma_Claudecode\figma-console-mcp\figma-desktop-bridge\manifest.json
   ```
3. With the target file open, run it: `Plugins → Development → Figma Desktop Bridge → Run`.
4. Its panel should show **Connected — AI …** — same as macOS.

> Note: the first time `npx figma-console-mcp@latest` runs, it will also auto-install its own bundled copy of this plugin under `%USERPROFILE%\.figma-console-mcp\plugin\manifest.json`. Either manifest (the repo one or the auto-installed one) works — you only need to import **one** of them into Figma, not both.

### 10. Verify the full chain on Windows

1. Confirm the local WebSocket server is listening (PowerShell equivalent of the macOS `lsof` check):
   ```powershell
   netstat -ano | findstr 9223
   ```
   or:
   ```powershell
   Test-NetConnection -ComputerName localhost -Port 9223
   ```
   `TcpTestSucceeded : True` means the server is up.

2. **Windows Defender Firewall** may prompt to allow `node.exe` to accept connections the first time the MCP server starts — click **Allow** (private networks is enough; it's a loopback-only connection to `localhost`, nothing external).

3. Open Antigravity in the project folder and ask it to check the Figma connection status — it should report success, matching what worked with Claude Code on macOS.

### Common Windows-specific gotchas

- **Port already in use**: if something else holds 9223, the plugin just falls back to 9224–9232 automatically — no action needed, but if you're manually checking with `netstat`, check that whole range.
- **Path separators**: always use the Windows path (`C:\...\manifest.json`) when importing into Figma Desktop — do not paste a macOS-style (`/Users/...`) path, it won't resolve.
- **Corporate VPN/proxy**: if `npx -y figma-console-mcp@1.40.0` hangs on first run, it's trying to download the package from the npm registry — check proxy/VPN settings block npmjs.org.
- **Antivirus**: some AV tools flag Node spawning child processes over a WebSocket port as suspicious on first run — this is the MCP server talking to the plugin, it's expected; whitelist the project folder / `node.exe` if it gets blocked.
- **Env var not picked up**: if Antigravity reports `FIGMA_ACCESS_TOKEN` missing/empty after step 8, the environment variable was set after Antigravity (or its terminal) was already open — fully quit and relaunch Antigravity, not just reload the window.
