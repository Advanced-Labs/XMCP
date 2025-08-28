# Browser-MCP Installation & Setup Guide

Complete installation guide for both the MCP server and browser extension components.

## Prerequisites

### Node.js & PNPM Setup

#### macOS/Linux (bash)
```bash
# Check Node version (prefer 18.x or 20.x LTS)
node -v

# Enable Corepack (preferred method)
corepack enable
corepack prepare pnpm@9.0.0 --activate

# Verify PNPM installation
pnpm -v
```

#### Windows (PowerShell)
```powershell
# Check Node version (prefer 18.x or 20.x LTS)
node -v

# Enable Corepack (preferred method)
corepack enable
corepack prepare pnpm@9.0.0 --activate

# Verify PNPM installation
pnpm -v
```

> **Note:** If `corepack` isn't found, update Node.js to version 16.19+ or install PNPM once with `npm i -g pnpm`, then prefer Corepack thereafter.

### Alternative PNPM Installation
If Corepack is not available:
```bash
# Via npm (one-time, then use Corepack)
npm install -g pnpm

# Via curl (macOS/Linux)
curl -fsSL https://get.pnpm.io/install.sh | sh

# Via PowerShell (Windows)
iwr https://get.pnpm.io/install.ps1 -useb | iex
```

## Repository Setup

### 1. Clone Repository
```bash
git clone https://github.com/djyde/browser-mcp.git
cd browser-mcp
```

### 2. Clean Previous Installations (Important!)
```bash
# Remove any mixed package manager artifacts
rm -rf node_modules
rm -rf **/node_modules
rm -f package-lock.json
rm -f yarn.lock

# Windows PowerShell alternative:
# Remove-Item -Recurse -Force node_modules, **/node_modules, package-lock.json, yarn.lock -ErrorAction SilentlyContinue
```

### 3. Install Dependencies
```bash
# Install all workspace dependencies at root
pnpm install
```

This command:
- Reads `pnpm-workspace.yaml` to find all packages
- Creates/updates a single `pnpm-lock.yaml` at the root
- Links local packages via `workspace:*` protocol
- Installs all dependencies using PNPM's content-addressable store

## Server Installation & Setup

### Development Mode
```bash
# Start server in development mode with hot reload
pnpm -C server dev

# Alternative using filter
pnpm --filter server dev
```

### Production Build
```bash
# Build the server
pnpm -C server build

# Run built server
node server/dist/cli.js
```

### Server Configuration

The server runs on **WebSocket port 11223** by default. You can verify it's running by checking the console output:
```
WebSocket server listening on port 11223
```

### MCP Client Configuration

To use with Claude Desktop, add this to your MCP configuration:

**macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
**Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

```json
{
  "mcpServers": {
    "browser-mcp": {
      "command": "node",
      "args": ["/path/to/browser-mcp/server/dist/cli.js"]
    }
  }
}
```

Or if installed globally:
```json
{
  "mcpServers": {
    "browser-mcp": {
      "command": "npx",
      "args": ["@djyde/mcp-browser@latest"]
    }
  }
}
```

## Extension Installation & Setup

### Building the Extension

#### For Chrome/Chromium
```bash
pnpm -C extension build
```

#### For Edge
```bash
pnpm -C extension build:edge
```

#### For Firefox
```bash
pnpm -C extension build:firefox
```

### Development Mode (Hot Reload)
```bash
# Chrome development mode
pnpm -C extension dev

# Firefox development mode
pnpm -C extension dev:firefox
```

### Loading the Extension

#### Chrome/Edge Installation
1. Open Chrome/Edge
2. Navigate to `chrome://extensions/` (Chrome) or `edge://extensions/` (Edge)
3. Enable "Developer mode" (toggle in top-right)
4. Click "Load unpacked"
5. Select the appropriate folder:
   - **Chrome:** `browser-mcp/extension/.output/chrome-mv3/`
   - **Edge:** `browser-mcp/extension/.output/edge-mv3/`

#### Firefox Installation
1. Open Firefox
2. Navigate to `about:debugging`
3. Click "This Firefox" in the left sidebar
4. Click "Load Temporary Add-on"
5. Navigate to `browser-mcp/extension/.output/firefox-mv3/`
6. Select the `manifest.json` file

### Extension Build Output Locations
- **Chrome:** `extension/.output/chrome-mv3/`
- **Edge:** `extension/.output/edge-mv3/`
- **Firefox:** `extension/.output/firefox-mv3/`

## Configuration & Settings

### Extension Permissions

The extension requires these permissions (automatically requested):
- **activeTab:** Access to current active tab
- **scripting:** Execute scripts in web pages
- **storage:** Store extension data

Optional permissions (requested when needed):
- **bookmarks:** Access browser bookmarks
- **history:** Search browser history

### WebSocket Configuration

The extension connects to `ws://localhost:11223` by default. This is configured in:
- `extension/entrypoints/background.ts`
- WebSocket server in `server/src/ws.ts`

To change the port, modify both files:

**Extension background.ts:**
```typescript
const ws = new WebSocket('ws://localhost:YOUR_PORT');
```

**Server ws.ts:**
```typescript
const wss = new WebSocketServer({ port: YOUR_PORT });
```

### Manifest Configuration

The extension uses **WXT framework** for building. Configuration is in `extension/wxt.config.ts`:

```typescript
export default defineConfig({
  modules: ['@wxt-dev/module-react'],
  manifest: {
    name: 'Browser MCP',
    permissions: ['activeTab', 'scripting', 'storage'],
    optional_permissions: ['bookmarks', 'history'],
    host_permissions: ['https://*/*'],
    web_accessible_resources: [{
      resources: ['*'],
      matches: ['<all_urls>']
    }]
  }
});
```

## Verification & Testing

### 1. Verify Server is Running
```bash
# Check if server starts without errors
pnpm -C server dev

# Should see:
# "WebSocket server listening on port 11223"
```

### 2. Verify Extension is Loaded
1. Check browser's extension management page
2. Confirm "Browser MCP" is enabled
3. Check for any errors in extension details

### 3. Test WebSocket Connection
1. Open browser DevTools (F12)
2. Go to Console
3. Load any webpage
4. Look for WebSocket connection logs (no errors)

### 4. Test MCP Integration
1. Open Claude Desktop (or your MCP client)
2. Try a basic command: `get_current_page_url`
3. Should return the current tab's URL

## Troubleshooting

### Common Issues

#### 1. "command not found: pnpm"
```bash
# Enable Corepack
corepack enable
corepack prepare pnpm@latest --activate

# Or install via npm
npm install -g pnpm
```

#### 2. WebSocket Connection Failed
- Ensure server is running: `pnpm -C server dev`
- Check port 11223 is not blocked by firewall
- Verify extension is loaded and enabled

#### 3. Extension Won't Load
- Check browser's extension management page for errors
- Ensure you selected the correct output folder:
  - Chrome: `extension/.output/chrome-mv3/`
  - Firefox: `extension/.output/firefox-mv3/`

#### 4. "Cannot resolve @browser-mcp/shared"
```bash
# Rebuild shared package
pnpm -C shared build
pnpm -C server build
pnpm -C extension build
```

#### 5. Mixed Package Manager Issues
```bash
# Clean all artifacts and reinstall
rm -rf node_modules **/node_modules package-lock.json yarn.lock
pnpm install
```

#### 6. Permission Errors in Browser
- Grant requested permissions when extension prompts
- Check `chrome://extensions/` → Details → Permissions
- Some features require user interaction first

### Development Debugging

#### Server Logs
```bash
# Server logs appear in terminal where you ran:
pnpm -C server dev
```

#### Extension Logs
1. Open browser DevTools (F12)
2. Go to Console
3. Extension logs appear here
4. For background script logs:
   - Chrome: `chrome://extensions/` → Details → Inspect views: background page
   - Firefox: `about:debugging` → This Firefox → Inspect

#### WebSocket Traffic
1. Browser DevTools → Network tab
2. Filter by "WS" (WebSocket)
3. Click WebSocket connection
4. View Messages tab for traffic

## Production Deployment

### Server Deployment
```bash
# Build for production
pnpm -C server build

# The built server is in server/dist/cli.js
# Deploy this file and run with: node cli.js
```

### Extension Publishing
```bash
# Create extension zip for store submission
pnpm -C extension zip          # Chrome/Edge zip
pnpm -C extension zip:firefox  # Firefox zip

# Zips are created in extension/.output/
```

### System Service (Linux)
Create a systemd service file at `/etc/systemd/system/browser-mcp.service`:

```ini
[Unit]
Description=Browser MCP Server
After=network.target

[Service]
Type=simple
User=your-user
WorkingDirectory=/path/to/browser-mcp
ExecStart=/usr/bin/node server/dist/cli.js
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl enable browser-mcp
sudo systemctl start browser-mcp
```

### Docker Deployment
Create `Dockerfile`:
```dockerfile
FROM node:18-alpine

WORKDIR /app
COPY . .

RUN corepack enable
RUN pnpm install
RUN pnpm -C server build

EXPOSE 11223

CMD ["node", "server/dist/cli.js"]
```

Build and run:
```bash
docker build -t browser-mcp .
docker run -p 11223:11223 browser-mcp
```

## Environment-Specific Notes

### Windows Considerations
- Use PowerShell or Windows Terminal (not CMD)
- Path separators use backslash `\` in some contexts
- Firewall may block WebSocket connections
- Windows Defender may flag WebSocket server

### macOS Considerations
- May need to allow network connections in Security preferences
- Use Terminal or iTerm2
- Ensure Node.js is installed via official installer or brew

### Linux Considerations
- Ensure Node.js 18+ is installed
- May need to install build tools: `sudo apt install build-essential`
- Check firewall rules for port 11223

This guide covers all installation scenarios and troubleshooting steps discovered during analysis of the browser-mcp project.