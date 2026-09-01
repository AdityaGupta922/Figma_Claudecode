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

## Windows setup (Antigravity) — start here, not from scratch

Code is already pushed: **https://github.com/AdityaGupta922/Figma_Claudecode** (private repo). On Windows you're only doing the pieces that can't carry over from macOS — installs, cloning, and re-registering the MCP server. Everything else (code, `package-lock.json`, the plugin source) comes with the clone.

**Install once:**
- Node.js **v26.x** ([nodejs.org](https://nodejs.org)) — repo has `.nvmrc` (`26.7.0`) + `engines` pin in `package.json`, so `npm install` won't silently drift versions.
- Git for Windows ([git-scm.com](https://git-scm.com/download/win))
- [Antigravity](https://antigravity.google/) (replaces Claude Code as the AI client here — same MCP server underneath)
- Figma Desktop for Windows ([figma.com/downloads](https://www.figma.com/downloads/))

**Get the code + deps:**
```powershell
git clone https://github.com/AdityaGupta922/Figma_Claudecode.git
cd Figma_Claudecode
npm install
```

**Get a Figma token:** Figma → Settings → Security → Personal access tokens → generate one for this machine (don't reuse the Mac one).

**Register the MCP server** — edit (create if missing) `C:\Users\<you>\.gemini\antigravity\mcp_config.json`:
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
Pinned to `1.40.0` (the exact version macOS runs) so behavior doesn't drift — not `@latest`. Then set the token as a real env var (don't paste it into the file):
```powershell
[System.Environment]::SetEnvironmentVariable("FIGMA_ACCESS_TOKEN", "<your-windows-token>", "User")
```
Restart Antigravity so it picks up both.

**Import the plugin into Figma Desktop:** `Plugins → Development → Import plugin from manifest…` → pick
`...\Figma_Claudecode\figma-console-mcp\figma-desktop-bridge\manifest.json` (already in the clone). Open the file, run the plugin (`Plugins → Development → Figma Desktop Bridge → Run`), confirm its panel shows **Connected**.

**Verify:** `netstat -ano | findstr 9223` should show a `LISTENING` line. Then ask Antigravity to check the Figma connection status — should report success.

**Gotchas:** Windows Defender may prompt to allow `node.exe` on first run — allow it. If the env var doesn't show up, you set it after Antigravity was already open — fully restart the app.
