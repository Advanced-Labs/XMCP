# Browser-MCP Command Development Guide

This guide covers everything you need to know about adding new commands to the browser-mcp system.

## Understanding the Command Flow

Commands in browser-mcp follow this flow:
```
MCP Client → Server (tools.ts) → WebSocket → Extension (calls.ts) → Browser APIs → Response back
```

## PNPM Workspace Workflow

### Prerequisites
```bash
# Enable corepack and activate PNPM
corepack enable
corepack prepare pnpm@9.0.0 --activate
pnpm -v

# Install dependencies at workspace root
pnpm install
```

### Development Commands
```bash
# Run server in development mode
pnpm -C server dev

# Build extension for Chrome
pnpm -C extension build

# Build extension for Firefox
pnpm -C extension build:firefox

# Build extension for Edge
pnpm -C extension build:edge

# Build all packages
pnpm -r build

# After editing shared package, rebuild it first
pnpm -C shared build
pnpm -C server build
pnpm -C extension build
```

## Adding a New Command: Step-by-Step

### Step 1: Add WebSocket Method to Shared Types

**File:** `shared/index.ts`

Add your method to the `WSMethods` enum:
```typescript
export enum WSMethods {
  GetCurrentTabUrl = "GetCurrentTabUrl",
  GetCurrentTabHtml = "GetCurrentTabHtml", 
  AppendStyle = "AppendStyle",
  ListBookmarks = "ListBookmarks",
  HistorySearch = "HistorySearch",
  YourNewMethod = "YourNewMethod"  // 👈 ADD HERE
}
```

### Step 2: Implement Extension Handler

**File:** `extension/calls.ts`

Add method to the `CallHandler` class:
```typescript
export class CallHandler {
  // ... existing methods ...

  async YourNewMethod(args: { param1: string; param2?: number }): Promise<string> {
    // Your browser interaction logic here
    // Use WebExtension APIs: chrome.tabs, chrome.storage, etc.
    
    // Example:
    const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
    
    // Return the result
    return `Processed: ${args.param1}`;
  }

  // ... rest of class
}
```

### Step 3: Add Tool Definition to Server

**File:** `server/src/tools.ts`

Add to the `Tools` enum:
```typescript
export enum Tools {
  GetCurrentPageUrl = "get_current_page_url",
  GetCurrentPageMarkdown = "get_current_page_markdown", 
  AppendStyle = "append_style",
  HistorySearch = "history_search",
  YourNewTool = "your_new_tool"  // 👈 ADD HERE
}
```

Add tool configuration to the `tools` array:
```typescript
export const tools: Tool[] = [
  // ... existing tools ...
  {
    name: Tools.YourNewTool,
    description: "Description of what your tool does", 
    inputSchema: {
      type: "object",
      properties: {
        param1: {
          type: "string",
          description: "Description of param1"
        },
        param2: {
          type: "number", 
          description: "Description of param2"
        }
      },
      required: ["param1"]  // List required parameters
    }
  }
];
```

### Step 4: Add Tool Handler Method

**File:** `server/src/tools.ts`

Add to the `ToolHandler` class:
```typescript
export class ToolHandler {
  // ... existing methods ...

  async [Tools.YourNewTool](args: { param1: string; param2?: number }) {
    const result = await call<string>(WSMethods.YourNewMethod, args);
    
    return {
      content: [
        {
          type: "text" as const,
          text: result
        }
      ]
    };
  }

  // ... rest of class
}
```

### Step 5: Build and Test

```bash
# Build shared package first (critical!)
pnpm -C shared build

# Build server and extension
pnpm -C server build
pnpm -C extension build

# Load extension in browser from:
# - Chrome: extension/.output/chrome-mv3/
# - Firefox: extension/.output/firefox-mv3/
# - Edge: extension/.output/edge-mv3/
```

## Command Development Patterns

### Simple Echo Command
```typescript
// Extension handler (calls.ts)
async Echo(args: { message: string }): Promise<string> {
  return `Echo: ${args.message}`;
}

// Server tool handler (tools.ts)
async [Tools.Echo](args: { message: string }) {
  const result = await call<string>(WSMethods.Echo, args);
  return {
    content: [{ type: "text" as const, text: result }]
  };
}
```

### Browser Interaction Command
```typescript
// Extension handler (calls.ts)
async GetPageTitle(args: {}): Promise<string> {
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
  return tab.title || "No title";
}
```

### Storage Interaction Command
```typescript
// Extension handler (calls.ts)
async SaveData(args: { key: string; value: string }): Promise<string> {
  await chrome.storage.local.set({ [args.key]: args.value });
  return `Saved ${args.key}`;
}
```

### Complex Return Types
```typescript
// Extension handler (calls.ts)
async GetTabInfo(args: {}): Promise<{ title: string; url: string; id: number }> {
  const [tab] = await chrome.tabs.query({ active: true, currentWindow: true });
  return {
    title: tab.title || "",
    url: tab.url || "",
    id: tab.id || 0
  };
}

// Server tool handler (tools.ts)
async [Tools.GetTabInfo](args: {}) {
  const result = await call<{ title: string; url: string; id: number }>(
    WSMethods.GetTabInfo, 
    args
  );
  
  return {
    content: [
      {
        type: "text" as const,
        text: `Title: ${result.title}\nURL: ${result.url}\nID: ${result.id}`
      }
    ]
  };
}
```

## Common Pitfalls & Solutions

### 1. Shared Package Not Built
**Problem:** Changes to shared types not reflected
**Solution:** Always `pnpm -C shared build` after editing shared/index.ts

### 2. Mixed Package Managers
**Problem:** Using npm/yarn in PNPM workspace
**Solution:** Remove package-lock.json/yarn.lock, use only PNPM commands

### 3. Extension Permissions
**Problem:** Chrome API calls fail with permission errors
**Solution:** Add required permissions to `extension/wxt.config.ts`:
```typescript
export default defineConfig({
  manifest: {
    permissions: ["activeTab", "storage", "tabs"], // Add as needed
  }
});
```

### 4. WebSocket Connection Issues
**Problem:** Extension can't connect to server
**Solution:** Ensure server is running on port 11223 and extension is loaded

### 5. Type Mismatches
**Problem:** TypeScript errors between server/extension
**Solution:** Use shared types from `@browser-mcp/shared` package

## Testing Your Commands

1. **Start the server:**
   ```bash
   pnpm -C server dev
   ```

2. **Load the extension** in browser developer mode

3. **Test via MCP client** (Claude Desktop, etc.):
   ```
   your_new_tool param1="test value" param2=42
   ```

4. **Check logs:**
   - Server logs in terminal
   - Extension logs in browser DevTools → Extensions → Inspect views

## Advanced Patterns

### Error Handling
```typescript
// Extension handler (calls.ts)
async RiskyOperation(args: { url: string }): Promise<string> {
  try {
    const response = await fetch(args.url);
    return await response.text();
  } catch (error) {
    throw new Error(`Failed to fetch: ${error.message}`);
  }
}
```

### Async Operations
```typescript
// Extension handler (calls.ts)
async LongRunningTask(args: { duration: number }): Promise<string> {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve(`Task completed after ${args.duration}ms`);
    }, args.duration);
  });
}
```

### Multiple Tab Operations
```typescript
// Extension handler (calls.ts)
async GetAllTabs(args: {}): Promise<string> {
  const tabs = await chrome.tabs.query({});
  const tabInfo = tabs.map(tab => `${tab.title}: ${tab.url}`);
  return tabInfo.join('\n');
}
```

## File Structure Recap

When adding a command, you'll edit these files:
- `shared/index.ts` - Add WSMethod enum
- `extension/calls.ts` - Add handler method
- `server/src/tools.ts` - Add tool definition and handler

Build order:
1. `pnpm -C shared build`
2. `pnpm -C server build` 
3. `pnpm -C extension build`

This ensures the workspace dependencies are properly linked and updated.