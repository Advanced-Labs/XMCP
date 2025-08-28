# Browser-MCP Project Blueprints

Complete architectural documentation of the browser-mcp project, including all components, files, and extensibility points.

## Project Overview

Browser-MCP is a **monorepo** that bridges AI assistants with web browsers through the Model Context Protocol (MCP). It enables AI assistants to interact with browser tabs, manipulate web pages, access browser APIs, and perform web-based tasks.

### Core Architecture
```
MCP Client (Claude Desktop) ←→ Server (stdio) ←→ WebSocket ←→ Browser Extension ←→ Browser APIs
```

### Repository Structure
```
browser-mcp/
├── pnpm-workspace.yaml         # PNPM workspace configuration
├── package.json                # Root package (private, no code)
├── pnpm-lock.yaml              # Lockfile for all packages
├── shared/                     # Shared types & utilities
├── server/                     # MCP server implementation
├── extension/                  # Browser extension (WXT)
└── docs/                       # Documentation
```

## Package Architecture

### PNPM Workspace Configuration

**File:** `pnpm-workspace.yaml`
```yaml
packages:
  - "shared"
  - "server"  
  - "extension"
```

**Root package.json:**
- **Type:** Private monorepo coordinator
- **Purpose:** Workspace-level scripts and devDependencies
- **Scripts:** Cross-package build orchestration
- **No runtime code**

## Shared Package (`shared/`)

### Purpose
Common types, interfaces, and utilities shared between server and extension.

### File Structure
```
shared/
├── package.json                # Package metadata
├── index.ts                    # Main export file
└── tsconfig.json               # TypeScript config
```

### Key Files Analysis

#### `shared/index.ts`
**Types Defined:**
```typescript
// WebSocket method enumeration
export enum WSMethods {
  GetCurrentTabUrl = "GetCurrentTabUrl",
  GetCurrentTabHtml = "GetCurrentTabHtml", 
  AppendStyle = "AppendStyle",
  ListBookmarks = "ListBookmarks",
  HistorySearch = "HistorySearch"
}

// Message type for method calls
export type MethodCall = {
  type: "call";
  id: string;
  method: WSMethods;
  args: Record<string, any>;
}

// Message type for method responses
export type MethodResponse = {
  type: "response";
  id: string;
  result: any;
  error?: string;
}
```

**What Can Be Changed:**
- ✅ Add new `WSMethods` enum values for new commands
- ✅ Extend message types with additional fields
- ✅ Add utility functions for message handling
- ❌ Don't change existing enum values (breaks compatibility)

#### `shared/package.json`
**Configuration:**
- **Name:** `@browser-mcp/shared`
- **Type:** ES module
- **Build target:** TypeScript → JavaScript/declarations
- **Dependencies:** Minimal (only dev dependencies)

## Server Package (`server/`)

### Purpose
MCP server that bridges between MCP clients and browser extension via WebSocket.

### File Structure  
```
server/
├── package.json                # Server dependencies & scripts
├── tsconfig.json               # TypeScript configuration
├── src/
│   ├── cli.ts                  # CLI entry point
│   ├── main.ts                 # MCP server setup
│   ├── tools.ts                # Tool definitions & handlers
│   └── ws.ts                   # WebSocket server
└── dist/                       # Built output
```

### Key Files Analysis

#### `server/src/main.ts`
**Purpose:** MCP server initialization and protocol handling

**Key Components:**
```typescript
// Server configuration
const server = new Server({
  name: "Browser MCP", 
  version: packageJson.version
}, {
  capabilities: {
    resources: {},
    logging: {},
    tools: {}
  }
});

// Request handlers
server.setRequestHandler(ListToolsRequestSchema, async () => ({
  tools: tools
}));

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  // Dynamic tool handler invocation
  return await toolHandler[request.params.name](request.params.arguments);
});

// Transport setup
const transport = new StdioServerTransport();
await server.connect(transport);
```

**What Can Be Changed:**
- ✅ Add new request handlers for MCP protocol extensions
- ✅ Modify server capabilities and metadata
- ✅ Add middleware for request/response processing
- ✅ Add logging and error handling
- ❌ Don't change core protocol message handling structure

#### `server/src/tools.ts`
**Purpose:** Tool definitions and business logic handlers

**Key Components:**
```typescript
// Tool enumeration
export enum Tools {
  GetCurrentPageUrl = "get_current_page_url",
  GetCurrentPageMarkdown = "get_current_page_markdown", 
  AppendStyle = "append_style",
  HistorySearch = "history_search"
}

// Tool configurations for MCP
export const tools: Tool[] = [
  {
    name: Tools.GetCurrentPageUrl,
    description: "Get the URL of the current active tab",
    inputSchema: {
      type: "object",
      properties: {},
      required: []
    }
  }
  // ... more tools
];

// Tool handler class
export class ToolHandler {
  constructor(private server: Server) {}

  async [Tools.GetCurrentPageUrl](args: {}) {
    const url = await call<string>(WSMethods.GetCurrentTabUrl, {});
    return {
      content: [{
        type: "text" as const,
        text: url
      }]
    };
  }
  // ... more handlers
}
```

**What Can Be Changed:**
- ✅ Add new tools to `Tools` enum
- ✅ Add new tool configurations to `tools` array
- ✅ Add new handler methods to `ToolHandler` class
- ✅ Modify input schemas for existing tools
- ✅ Change response formatting
- ⚠️ Tool names should follow kebab-case convention
- ❌ Don't remove existing tools without deprecation

#### `server/src/ws.ts`
**Purpose:** WebSocket server for extension communication

**Key Components:**
```typescript
// WebSocket server setup
const wss = new WebSocketServer({ port: 11223 });

// Single connection management
let currentWS: WebSocket | null = null;

// Method call with Promise-based response handling
export async function call<T = any>(method: WSMethods, args: any): Promise<T> {
  return new Promise((resolve, reject) => {
    const id = nanoid();
    const message: MethodCall = { type: "call", id, method, args };
    
    // Store pending request
    pendingRequests.set(id, { resolve, reject });
    
    // Send via WebSocket
    currentWS?.send(JSON.stringify(message));
  });
}

// Response handling
ws.on('message', (data) => {
  const response: MethodResponse = JSON.parse(data.toString());
  const pending = pendingRequests.get(response.id);
  
  if (pending) {
    if (response.error) {
      pending.reject(new Error(response.error));
    } else {
      pending.resolve(response.result);
    }
    pendingRequests.delete(response.id);
  }
});
```

**What Can Be Changed:**
- ✅ Change WebSocket port (update extension too)
- ✅ Add connection authentication
- ✅ Add message encryption
- ✅ Add connection pooling for multiple extensions
- ✅ Add request timeout handling
- ✅ Add message queuing for disconnected clients
- ❌ Don't change core message format without updating shared types

#### `server/src/cli.ts`
**Purpose:** CLI entry point for the server

**Simple wrapper that calls main:**
```typescript
import { main } from './main.js';

main().catch(console.error);
```

**What Can Be Changed:**
- ✅ Add CLI argument parsing
- ✅ Add configuration file loading
- ✅ Add process signal handlers
- ✅ Add daemon mode support

### Server Dependencies

#### Production Dependencies
- **@modelcontextprotocol/sdk:** Core MCP protocol implementation
- **ws:** WebSocket server library
- **nanoid:** Unique ID generation for request tracking
- **zod:** Runtime type validation
- **@mozilla/readability:** Web page content extraction
- **turndown:** HTML to Markdown conversion
- **jsdom:** Server-side DOM manipulation

#### Development Dependencies
- **@browser-mcp/shared:** Local workspace dependency
- **TypeScript & types:** Build toolchain

## Extension Package (`extension/`)

### Purpose
Browser extension that executes browser interactions and communicates with server via WebSocket.

### File Structure
```
extension/
├── package.json                # Extension dependencies & build scripts
├── wxt.config.ts               # WXT framework configuration
├── tsconfig.json               # TypeScript configuration
├── calls.ts                    # WebSocket method handlers
├── storage.ts                  # Extension storage utilities
├── entrypoints/
│   ├── background.ts           # Background service worker
│   └── popup/                  # Extension popup UI
├── assets/                     # Static assets (icons, etc.)
├── public/                     # Public assets
├── utils/                      # Utility functions
└── .output/                    # Built extension (gitignored)
    ├── chrome-mv3/             # Chrome extension build
    ├── firefox-mv3/            # Firefox extension build
    └── edge-mv3/               # Edge extension build
```

### Key Files Analysis

#### `extension/wxt.config.ts`
**Purpose:** WXT framework build configuration

**Configuration:**
```typescript
export default defineConfig({
  modules: ['@wxt-dev/module-react'],
  vite: {
    plugins: [tailwindcss()]
  },
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

**What Can Be Changed:**
- ✅ Add new permissions for additional browser APIs
- ✅ Modify build targets (Chrome, Firefox, Edge)
- ✅ Add new Vite plugins for build processing
- ✅ Change manifest version or metadata
- ✅ Add content security policy rules
- ⚠️ Permissions changes require user consent

#### `extension/entrypoints/background.ts`
**Purpose:** Background service worker managing WebSocket connection

**Key Components:**
```typescript
// WebSocket connection management
function makeConnection() {
  const ws = new WebSocket('ws://localhost:11223');
  
  ws.onopen = () => {
    connectStatusStorage.setValue(true);
  };
  
  ws.onclose = () => {
    connectStatusStorage.setValue(false);
    // Reconnection logic
  };
  
  ws.onmessage = async (event) => {
    const message = JSON.parse(event.data);
    if (message.type === 'call') {
      try {
        const result = await callHandler.handle(message.method, message.args);
        ws.send(JSON.stringify({
          type: 'response',
          id: message.id,
          result
        }));
      } catch (error) {
        ws.send(JSON.stringify({
          type: 'response',
          id: message.id,
          error: error.message
        }));
      }
    }
  };
}

// Keep-alive for service worker
setInterval(() => {
  if (ws?.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({ type: 'keepalive' }));
  }
}, 5000);
```

**What Can Be Changed:**
- ✅ Modify WebSocket URL/port to match server
- ✅ Add connection retry logic with exponential backoff
- ✅ Add message queuing for offline scenarios
- ✅ Add connection authentication
- ✅ Modify keep-alive interval
- ⚠️ Service workers have limited lifetime in MV3

#### `extension/calls.ts`
**Purpose:** WebSocket method handler implementations

**Key Components:**
```typescript
export class CallHandler {
  async handle(method: string, args: any): Promise<any> {
    // Dynamic method dispatch
    if (typeof this[method] === 'function') {
      return await this[method](args);
    }
    throw new Error(`Method ${method} not found`);
  }

  async GetCurrentTabUrl(): Promise<string> {
    const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
    return tab.url || '';
  }

  async GetCurrentTabHtml(): Promise<string> {
    const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
    
    if (!tab.id) throw new Error('No active tab');
    
    const [result] = await chrome.scripting.executeScript({
      target: { tabId: tab.id },
      func: () => document.documentElement.outerHTML,
    });
    
    return result.result;
  }

  async AppendStyle(args: { css: string }): Promise<string> {
    const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
    
    if (!tab.id) throw new Error('No active tab');
    
    await chrome.scripting.insertCSS({
      target: { tabId: tab.id },
      css: args.css,
    });
    
    return 'Style appended successfully';
  }

  async HistorySearch(args: { query: string }): Promise<string> {
    const results = await chrome.history.search({
      text: args.query,
      maxResults: 10
    });
    
    return results.map(item => `${item.title}: ${item.url}`).join('\n');
  }
}
```

**What Can Be Changed:**
- ✅ Add new method handlers for additional browser APIs
- ✅ Modify existing handlers to change behavior
- ✅ Add error handling and validation
- ✅ Add caching for expensive operations
- ✅ Add permission checking before API calls
- ⚠️ Method names must match `WSMethods` enum in shared package
- ⚠️ Some APIs require user interaction or additional permissions

#### `extension/storage.ts`
**Purpose:** Extension storage utilities

**Utilities for:**
- Connection status tracking
- Extension configuration
- Cached data management

**What Can Be Changed:**
- ✅ Add new storage schemas
- ✅ Add data migration logic
- ✅ Add storage encryption
- ✅ Add storage quota management

### Extension Dependencies

#### Production Dependencies
- **@browser-mcp/shared:** Local workspace dependency
- **@wxt-dev/storage:** Extension storage utilities
- **@webext-core/proxy-service:** Service worker communication
- **React ecosystem:** UI framework (popup)
- **Tailwind CSS:** Styling framework

#### Development Dependencies
- **WXT:** Extension build framework
- **TypeScript:** Build toolchain
- **@types/chrome:** Chrome extension API types

## Communication Protocol

### Message Flow Architecture
```
1. MCP Client sends tool call to Server
2. Server validates and forwards via WebSocket to Extension
3. Extension executes browser API calls
4. Extension returns result via WebSocket to Server
5. Server formats response for MCP Client
```

### WebSocket Message Types

#### Method Call (Server → Extension)
```typescript
{
  type: "call",
  id: "unique-request-id",
  method: "GetCurrentTabUrl", 
  args: { param1: "value1" }
}
```

#### Method Response (Extension → Server)
```typescript
// Success
{
  type: "response",
  id: "unique-request-id", 
  result: "https://example.com"
}

// Error
{
  type: "response",
  id: "unique-request-id",
  error: "Permission denied"
}
```

#### Keep-alive (Extension → Server)
```typescript
{
  type: "keepalive"
}
```

## Extensibility Points

### Adding New Browser APIs

**1. Define WebSocket Method** (`shared/index.ts`):
```typescript
export enum WSMethods {
  // ... existing methods
  NewBrowserAPI = "NewBrowserAPI"
}
```

**2. Implement Extension Handler** (`extension/calls.ts`):
```typescript
async NewBrowserAPI(args: { param: string }): Promise<string> {
  // Use chrome.* APIs here
  const result = await chrome.someAPI.method(args.param);
  return result;
}
```

**3. Create MCP Tool** (`server/src/tools.ts`):
```typescript
export enum Tools {
  // ... existing tools
  NewBrowserTool = "new_browser_tool"
}

// Add to tools array and ToolHandler class
```

### Permission Requirements by API

**No Additional Permissions:**
- `chrome.tabs.query()` with basic filters
- `chrome.runtime.*` APIs
- Basic storage operations

**Requires `activeTab`:**
- `chrome.scripting.executeScript()`
- `chrome.scripting.insertCSS()`
- Content script injection

**Requires `tabs`:**
- Full tab information access
- Cross-origin tab manipulation

**Requires `bookmarks`:**
- `chrome.bookmarks.*` APIs

**Requires `history`:**
- `chrome.history.*` APIs

**Requires `cookies`:**
- `chrome.cookies.*` APIs

**Requires Host Permissions:**
- Cross-origin fetch requests
- Content script injection on specific domains

### Browser API Categories

#### Tab Management
- Query tabs: `chrome.tabs.query()`
- Create/close tabs: `chrome.tabs.create()`, `chrome.tabs.remove()`
- Navigate tabs: `chrome.tabs.update()`
- Tab events: `chrome.tabs.onUpdated`

#### Content Manipulation
- Execute scripts: `chrome.scripting.executeScript()`
- Insert CSS: `chrome.scripting.insertCSS()`
- Register content scripts: `chrome.scripting.registerContentScripts()`

#### Data Access
- Storage: `chrome.storage.local`, `chrome.storage.sync`
- Bookmarks: `chrome.bookmarks.*`
- History: `chrome.history.*` 
- Cookies: `chrome.cookies.*`

#### Network & Requests
- Web requests: `chrome.webRequest.*` (requires webRequest permission)
- Declarative net request: `chrome.declarativeNetRequest.*`

#### UI Integration
- Context menus: `chrome.contextMenus.*`
- Omnibox: `chrome.omnibox.*`
- Notifications: `chrome.notifications.*`

## Build System Architecture

### PNPM Workspace Build Flow
```
1. pnpm install (root) → Links all packages
2. pnpm -C shared build → Builds shared types
3. pnpm -C server build → Builds server with shared deps
4. pnpm -C extension build → Builds extension with shared deps
```

### Build Tools by Package

#### Shared Package
- **TypeScript Compiler:** `tsc` for declaration generation
- **Output:** `dist/` with `.js` and `.d.ts` files

#### Server Package  
- **tsup:** Fast TypeScript bundler
- **Target:** Node.js ESM
- **Output:** `dist/cli.js` (single file)

#### Extension Package
- **WXT:** Extension-specific bundler
- **Vite:** Underlying build tool
- **Targets:** Chrome MV3, Firefox MV3, Edge MV3
- **Output:** Browser-specific builds in `.output/`

### Development vs Production Builds

#### Development
- **Server:** `tsx` or `ts-node` for direct TS execution
- **Extension:** WXT dev server with hot reload
- **Shared:** Watch mode compilation

#### Production  
- **Server:** Bundled single file for deployment
- **Extension:** Optimized builds for store submission
- **Shared:** Compiled with declarations for consumption

## Security Considerations

### Extension Security
- **Content Security Policy:** Configured in manifest
- **Host permissions:** Minimal required set
- **Sandboxing:** Extension runs in isolated context
- **Cross-origin:** Controlled via host_permissions

### WebSocket Security
- **Local connection only:** `ws://localhost:11223`
- **No authentication:** Assumes local server trust
- **Message validation:** JSON parsing with error handling
- **Rate limiting:** Consider adding for production

### MCP Protocol Security
- **Stdio transport:** Process-level isolation
- **Input validation:** Server validates all tool parameters
- **Error handling:** No sensitive data in error messages

## Performance Considerations

### WebSocket Connection
- **Single connection:** One persistent WebSocket
- **Keep-alive:** Prevents service worker dormancy
- **Reconnection:** Automatic retry with backoff
- **Message queuing:** Handle temporary disconnections

### Browser API Efficiency
- **Tab queries:** Cache active tab when possible
- **Script execution:** Minimize injected script size
- **Storage:** Use appropriate storage type (local/sync)
- **Permissions:** Request only when needed

### Memory Management
- **Service worker lifecycle:** Handle dormancy in MV3
- **Event listeners:** Proper cleanup to prevent leaks
- **Large responses:** Stream or chunk large data
- **Caching:** Cache expensive operations locally

This blueprint provides comprehensive understanding of every component, file, and extensibility point in the browser-mcp project, enabling informed development decisions and system modifications.