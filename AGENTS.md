# Agent Instructions for Convex with Svelte

This is your comprehensive guide for working with Convex in Svelte/SvelteKit projects.

## Overview

You have access to three powerful MCP servers:
1. **Convex MCP Server** - For interacting with Convex deployments
2. **Svelte MCP Server** - For accessing Svelte 5 and SvelteKit documentation
3. **Bits UI Documentation** - For headless component primitives (fetch on demand)

## Priority Rules

**CRITICAL: When working with Convex and Svelte together:**
1. **Always prefer `convex-svelte` patterns** for frontend integration
2. **Always use Convex MCP tools** over command-line tools where possible
3. **Follow Svelte 5 runes and reactive patterns** when writing frontend code
4. **Use proper validators and TypeScript types** in Convex backend functions
5. **Use Bits UI for headless components** - fetch docs on demand when building UI

---

## Convex MCP Server Tools

### Deployment Tools

#### `mcp_convex-mcp_status`
**When to use:** Always use this FIRST when working with Convex deployments.

**Purpose:** Queries available deployments and returns deployment selectors needed for all other tools.

**Important:**
- Projects have `dev` (development) and `prod` (production) deployments
- **Default to `dev` deployment** unless specifically debugging production
- Local projects have a single `urlWithAdminKey` deployment
- The deployment selector returned is required for all other MCP tools

**Example workflow:**
```
1. Call status to get deployment selector
2. Use deployment selector with other tools (tables, data, run, etc.)
```

---

### Table Tools

#### `mcp_convex-mcp_tables`
**Purpose:** Lists all tables with their declared and inferred schemas.

**When to use:**
- Understanding database structure
- Before writing queries or mutations
- Checking what tables exist
- Viewing schema validators

#### `mcp_convex-mcp_data`
**Purpose:** Paginate through documents in a table.

**Parameters:**
- `tableName`: Name of the table
- `order`: "asc" or "desc" 
- `limit`: Max results (default 100, max 1000)
- `cursor`: For pagination

**When to use:**
- Inspecting table data
- Debugging data issues
- Understanding document structure

#### `mcp_convex-mcp_runOneoffQuery`
**Purpose:** Execute sandboxed read-only JavaScript queries.

**Template:**
```js
import { query } from "convex:/_system/repl/wrappers.js";

export default query({
  handler: async (ctx) => {
    // Your query code here
    // Can only read data, cannot modify
  },
});
```

**When to use:**
- One-off data exploration
- Testing query logic
- Debugging without writing files

**Limitations:**
- Read-only (no database modifications)
- No external imports available
- Completely sandboxed

---

### Function Tools

#### `mcp_convex-mcp_functionSpec`
**Purpose:** Get metadata for all deployed functions.

**Returns for each function:**
- Function identifier (path + exported name)
- Argument validators
- Return value validators
- Type (query, mutation, action)
- Visibility (public or internal)

**When to use:**
- Understanding available API endpoints
- Before calling functions
- Checking function signatures

#### `mcp_convex-mcp_run`
**Purpose:** Execute deployed Convex functions.

**Parameters:**
- `deploymentSelector`: From status tool
- `functionName`: e.g., 'messages.send'
- `args`: JSON-encoded argument object

**When to use:**
- Testing functions
- Triggering mutations or actions
- Debugging function behavior

**Example:**
```typescript
// Running api.messages.send
await mcp_convex_run({
  deploymentSelector: devDeployment,
  functionName: "messages.send",
  args: JSON.stringify({ author: "Alice", body: "Hello" })
});
```

#### `mcp_convex-mcp_logs`
**Purpose:** Fetch recent function execution logs.

**Parameters:**
- `deploymentSelector`: From status tool
- `cursor`: Optional cursor for pagination (in ms)
- `entriesLimit`: Max entries (1-1000)
- `jsonl`: Boolean - true for JSONL, false for formatted text
- `tokensLimit`: Approximate token limit (default 20000)

**When to use:**
- Debugging function execution
- Checking console.log output
- Understanding execution flow
- Investigating errors

**Important:**
- Does not tail - performs single fetch
- Use cursor for pagination through history
- Similar to `bunx convex logs` but returns structured data

---

### Environment Variable Tools

#### `mcp_convex-mcp_envList`
List all environment variables in deployment.

#### `mcp_convex-mcp_envGet`
Get specific environment variable value.

#### `mcp_convex-mcp_envSet`
Set or update environment variable.

#### `mcp_convex-mcp_envRemove`
Remove environment variable.

**When to use:**
- Managing API keys
- Configuring deployment settings
- Debugging environment-specific issues

---

## Convex + Svelte Integration

### Installation

```bash
bun install convex convex-svelte
bunx convex init
```

### Core Patterns

#### 1. Setup Convex Client

In your root layout (`src/routes/+layout.svelte`):

```svelte
<script>
  import { setupConvex } from 'convex-svelte';
  import { PUBLIC_CONVEX_URL } from '$env/static/public';

  setupConvex(PUBLIC_CONVEX_URL);
</script>

<slot />
```

#### 2. Query Data with useQuery

```svelte
<script lang="ts">
  import { useQuery } from 'convex-svelte';
  import { api } from '../convex/_generated/api.js';

  const messages = useQuery(api.messages.list, () => ({ channelId }));
</script>

{#if messages.isLoading}
  Loading...
{:else if messages.error}
  Error: {messages.error}
{:else}
  {#each messages.data as message}
    <div>{message.content}</div>
  {/each}
{/if}
```

**Key Points:**
- `useQuery` returns `{ data, isLoading, error }`
- First arg: function reference from `api` object
- Second arg: function returning arguments object
- Third arg (optional): options like `initialData`, `keepPreviousData`

#### 3. Call Mutations

```svelte
<script lang="ts">
  import { useConvexClient } from 'convex-svelte';
  import { api } from '../convex/_generated/api.js';

  const client = useConvexClient();

  let message = $state('');

  function handleSubmit(event: SubmitEvent) {
    event.preventDefault();
    
    client.mutation(api.messages.send, {
      body: message,
      authorId: userId
    });
    
    message = '';
  }
</script>

<form onsubmit={handleSubmit}>
  <input bind:value={message} />
  <button type="submit">Send</button>
</form>
```

#### 4. Call Actions

```svelte
<script lang="ts">
  import { useConvexClient } from 'convex-svelte';
  import { api } from '../convex/_generated/api.js';

  const client = useConvexClient();

  async function generateResponse() {
    const result = await client.action(api.ai.generateResponse, {
      prompt: userInput
    });
    console.log(result);
  }
</script>
```

#### 5. Conditionally Skip Queries

Return `'skip'` from the arguments function to skip a query:

```svelte
<script lang="ts">
  import { useQuery } from 'convex-svelte';
  import { api } from '../convex/_generated/api.js';

  let auth = $state({ isAuthenticated: false });

  const user = useQuery(
    api.users.getActiveUser,
    () => auth.isAuthenticated ? {} : 'skip'
  );
</script>

{#if user.isLoading}
  Loading...
{:else if user.data}
  Welcome, {user.data.name}!
{/if}
```

**When query is skipped:**
- `isLoading` = `false`
- `error` = `null`
- `data` = `undefined`

#### 6. Server-Side Rendering (SSR)

Use `initialData` to avoid loading states:

**+page.server.ts:**
```typescript
import { ConvexHttpClient } from 'convex/browser';
import { PUBLIC_CONVEX_URL } from '$env/static/public';
import { api } from '../convex/_generated/api.js';
import type { PageServerLoad } from './$types.js';

export const load = (async () => {
  const client = new ConvexHttpClient(PUBLIC_CONVEX_URL);
  return {
    messages: await client.query(api.messages.list, { channelId: '123' })
  };
}) satisfies PageServerLoad;
```

**+page.svelte:**
```svelte
<script lang="ts">
  import { useQuery } from 'convex-svelte';
  import { api } from '../convex/_generated/api.js';
  import type { PageData } from './$types.js';

  let { data }: { data: PageData } = $props();

  const messages = useQuery(
    api.messages.list,
    () => ({ channelId: '123' }),
    () => ({ initialData: data.messages })
  );
</script>
```

---

## Convex Backend Patterns

### Function Definition

**Always use new function syntax:**

```typescript
import { query, mutation, action } from "./_generated/server";
import { v } from "convex/values";

export const myQuery = query({
  args: { userId: v.id("users") },
  returns: v.object({ name: v.string() }),
  handler: async (ctx, args) => {
    const user = await ctx.db.get(args.userId);
    return { name: user.name };
  },
});
```

### Internal vs Public Functions

```typescript
// Public (exposed to Internet)
import { query, mutation, action } from "./_generated/server";

// Internal (only callable by other Convex functions)
import { internalQuery, internalMutation, internalAction } from "./_generated/server";
```

**Important:**
- Use `internalQuery`, `internalMutation`, `internalAction` for private functions
- Use `query`, `mutation`, `action` for public API functions
- Internal functions can only be called by other Convex functions

### Function References

```typescript
// Import in any Convex function
import { api, internal } from "./_generated/api";

// Call public function
await ctx.runQuery(api.messages.list, { channelId });

// Call internal function
await ctx.runMutation(internal.messages.deleteOld, {});
```

**File-based routing:**
- Function in `convex/example.ts` named `f` → `api.example.f`
- Internal function in `convex/example.ts` named `g` → `internal.example.g`
- Function in `convex/messages/access.ts` named `h` → `api.messages.access.h`

### Validators

```typescript
import { v } from "convex/values";

// Common validators
v.string()
v.number()
v.boolean()
v.null()
v.id("tableName")
v.array(v.string())
v.object({ name: v.string(), age: v.number() })
v.union(v.string(), v.number())
v.optional(v.string())
v.record(v.string(), v.number()) // For dynamic keys
v.literal("value") // For discriminated unions
v.int64() // For bigint
v.bytes() // For ArrayBuffer

// Return validators are ALWAYS required
returns: v.null() // If function returns nothing
```

**Important:**
- Always include `args` and `returns` validators for all functions
- Use `v.null()` for functions that don't return anything
- Use `v.id("tableName")` for document IDs, not `v.string()`

### Schema Definition

**Always define schema in `convex/schema.ts`:**

```typescript
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

export default defineSchema({
  messages: defineTable({
    channelId: v.id("channels"),
    authorId: v.optional(v.id("users")),
    content: v.string(),
    createdAt: v.number(),
  })
    .index("by_channel", ["channelId"])
    .index("by_channel_and_time", ["channelId", "createdAt"]),
});
```

**Important:**
- System fields `_id` and `_creationTime` are automatic
- Index names should include all fields: "by_field1_and_field2"
- Query fields in same order as index
- Field names must be nonempty and not start with "$" or "_"

### Queries

**Use indexes, not filters:**

```typescript
// ❌ BAD (slow table scan)
const messages = await ctx.db
  .query("messages")
  .filter(q => q.eq(q.field("channelId"), channelId))
  .collect();

// ✅ GOOD (uses index)
const messages = await ctx.db
  .query("messages")
  .withIndex("by_channel", q => q.eq("channelId", channelId))
  .collect();
```

**Ordering:**

```typescript
// Default: ascending by _creationTime
const messages = await ctx.db.query("messages").collect();

// Explicit ordering
const messages = await ctx.db
  .query("messages")
  .withIndex("by_channel", q => q.eq("channelId", channelId))
  .order("desc")
  .take(10);
```

**Important:**
- Use `.collect()` to get all results as array
- Use `.take(n)` to limit results
- Use `.unique()` to get single result (throws if multiple match)
- For async iteration: `for await (const row of query)` - don't use `.collect()` or `.take()`

### Mutations

```typescript
// Insert
const id = await ctx.db.insert("messages", {
  channelId,
  content: "Hello"
});

// Patch (shallow merge)
await ctx.db.patch(messageId, { content: "Updated" });

// Replace (full replacement)
await ctx.db.replace(messageId, {
  channelId,
  content: "Replaced",
  // All required fields must be included
});

// Delete
await ctx.db.delete(messageId);
```

**Important:**
- `patch` only updates specified fields
- `replace` requires all required fields
- Both throw if document doesn't exist

### Actions

```typescript
// Use for external API calls, long-running operations
"use node"; // Add at top if using Node.js modules

import { action } from "./_generated/server";
import { v } from "convex/values";

export const generateAI = action({
  args: { prompt: v.string() },
  returns: v.string(),
  handler: async (ctx, args) => {
    // Call external APIs
    const response = await fetch("https://api.example.com", {
      method: "POST",
      body: JSON.stringify({ prompt: args.prompt }),
    });
    
    const data = await response.json();
    
    // Save result via mutation
    await ctx.runMutation(api.messages.save, {
      content: data.result
    });
    
    return data.result;
  },
});
```

**Important:**
- Actions run in Node.js runtime (queries/mutations run in V8)
- Actions cannot use `ctx.db` directly
- Use `ctx.runQuery` and `ctx.runMutation` to access database
- Add `"use node";` at top for Node.js built-in modules
- Add `@types/node` to package.json when using Node.js modules

### Function Calling

```typescript
// Use ctx.runQuery to call a query from a query, mutation, or action
await ctx.runQuery(api.messages.get, { id });

// Use ctx.runMutation to call a mutation from a mutation or action
await ctx.runMutation(api.messages.create, { content });

// Use ctx.runAction to call an action from an action
await ctx.runAction(api.external.fetch, { url });
```

**Important:**
- All calls take a `FunctionReference` - don't pass the function directly
- When calling function in same file, add type annotation to avoid TypeScript circularity:
  ```typescript
  const result: string = await ctx.runQuery(api.example.f, { name: "Bob" });
  ```

### Pagination

```typescript
import { paginationOptsValidator } from "convex/server";

export const listMessages = query({
  args: { 
    channelId: v.id("channels"),
    paginationOpts: paginationOptsValidator 
  },
  returns: v.object({
    page: v.array(v.object({ /* message shape */ })),
    isDone: v.boolean(),
    continueCursor: v.string(),
  }),
  handler: async (ctx, args) => {
    return await ctx.db
      .query("messages")
      .withIndex("by_channel", q => q.eq("channelId", args.channelId))
      .order("desc")
      .paginate(args.paginationOpts);
  },
});
```

**paginationOpts properties:**
- `numItems`: max documents to return (validator: `v.number()`)
- `cursor`: cursor for next page (validator: `v.union(v.string(), v.null())`)

**Returns:**
- `page`: array of documents
- `isDone`: boolean indicating if this is last page
- `continueCursor`: string cursor for next page

**In Svelte:**

```svelte
<script lang="ts">
  let paginationOpts = $state({ numItems: 20, cursor: null });
  
  const messages = useQuery(
    api.messages.listMessages,
    () => ({ channelId, paginationOpts })
  );
  
  function loadMore() {
    if (!messages.data?.isDone) {
      paginationOpts = {
        numItems: 20,
        cursor: messages.data.continueCursor
      };
    }
  }
</script>
```

### Cron Jobs

```typescript
import { cronJobs } from "convex/server";
import { internal } from "./_generated/api";
import { internalAction } from "./_generated/server";

const cleanupOldData = internalAction({
  args: {},
  returns: v.null(),
  handler: async (ctx, args) => {
    // Cleanup logic
    return null;
  },
});

const crons = cronJobs();

// Run every 2 hours
crons.interval("cleanup old data", { hours: 2 }, internal.crons.cleanupOldData, {});

// Or use cron syntax
crons.cron("daily cleanup", "0 0 * * *", internal.crons.cleanupOldData, {});

export default crons;
```

**Important:**
- Only use `crons.interval` or `crons.cron` methods
- Both take a `FunctionReference` - don't pass function directly
- Always import `internal` from `_generated/api`, even for functions in same file
- Export `crons` as default

### File Storage

```typescript
// Get signed URL for file
const url = await ctx.storage.getUrl(fileId); // Returns null if not found

// Get file metadata by querying _storage system table
import { Id } from "./_generated/dataModel";

type FileMetadata = {
  _id: Id<"_storage">;
  _creationTime: number;
  contentType?: string;
  sha256: string;
  size: number;
}

const metadata: FileMetadata | null = await ctx.db.system.get(fileId);
```

**Important:**
- Don't use deprecated `ctx.storage.getMetadata()` - query `_storage` table instead
- Storage stores items as `Blob` objects - convert to/from Blob when using

### HTTP Endpoints

**Always define in `convex/http.ts`:**

```typescript
import { httpRouter } from "convex/server";
import { httpAction } from "./_generated/server";

const http = httpRouter();

http.route({
  path: "/api/webhook",
  method: "POST",
  handler: httpAction(async (ctx, req) => {
    const body = await req.json();
    // Process webhook
    return new Response(JSON.stringify({ success: true }), {
      status: 200,
      headers: { "Content-Type": "application/json" }
    });
  }),
});

export default http;
```

**Important:**
- Endpoints registered at exact path specified
- Use `httpAction` decorator for handlers

### Full Text Search

```typescript
const messages = await ctx.db
  .query("messages")
  .withSearchIndex("search_body", (q) =>
    q.search("body", "hello hi").eq("channel", "#general")
  )
  .take(10);
```

### TypeScript Types

```typescript
import { Doc, Id } from "./_generated/dataModel";

// Get document type
type User = Doc<"users">;

// Get ID type
type UserId = Id<"users">;

// Use in function
export const getUser = query({
  args: { userId: v.id("users") },
  returns: v.object({ /* user shape */ }),
  handler: async (ctx, args) => {
    const userId: Id<"users"> = args.userId;
    const user: Doc<"users"> | null = await ctx.db.get(userId);
    return user;
  },
});

// Record with ID keys
const idToUsername: Record<Id<"users">, string> = {};
```

**Important:**
- Be strict with types, especially around IDs
- Use `Id<"tableName">` not `string`
- Always use `as const` for string literals in discriminated unions
- Define arrays as `const array: Array<T> = [...]`
- Define records as `const record: Record<K, V> = {...}`

---

## Svelte MCP Server Tools

### 1. `mcp_svelte_list-sections`

**When to use:** ALWAYS call this FIRST for any Svelte question.

**Purpose:** Lists all available Svelte 5 and SvelteKit documentation sections with use_cases.

**Workflow:**
1. User asks about Svelte/SvelteKit
2. Call `list-sections` 
3. Analyze `use_cases` field to find relevant sections
4. Call `get-documentation` with ALL relevant sections

### 2. `mcp_svelte_get-documentation`

**Purpose:** Retrieve full documentation content for sections.

**Parameters:**
- `section`: Single section name or array of section names

**Important:**
- Can search by title (e.g., "$state", "routing")
- Can search by file path (e.g., "cli/overview")
- **Fetch ALL relevant sections at once** (accepts arrays)

**Example:**
```typescript
// User wants to build interactive form
// After analyzing list-sections, fetch:
mcp_svelte_get_documentation({
  section: ["$state", "$derived", "$effect", "forms", "form-actions"]
});
```

### 3. `mcp_svelte_svelte-autofixer`

**Purpose:** Analyze Svelte code for issues and suggestions.

**CRITICAL RULE:** You MUST use this tool whenever writing Svelte code before sending it to the user.

**Workflow:**
1. Write Svelte component
2. Call `svelte-autofixer`
3. Apply suggestions
4. Call again until no issues
5. Only then send to user

**Parameters:**
- `code`: The Svelte component code
- `desired_svelte_version`: Version (default to 5)
- `filename`: Optional - component name with .svelte extension
- `async`: Boolean - true if component uses await

### 4. `mcp_svelte_playground-link`

**Purpose:** Generate Svelte Playground link.

**When to use:**
- After completing code
- Only after asking user if they want it
- NEVER if code was written to project files

**Parameters:**
- `name`: Playground name (reflect user task)
- `tailwind`: Boolean - only true if using Tailwind classes
- `files`: Object with filename keys and content values

**Example:**
```typescript
mcp_svelte_playground_link({
  name: "Interactive Form Example",
  tailwind: false,
  files: {
    'App.svelte': '<script>...</script>...',
    'utils.js': 'export function ...'
  }
});
```

---

## Bits UI Documentation

### What is Bits UI?

Bits UI is a headless component library for Svelte that provides accessible, unstyled component primitives. It's perfect for building custom UI components with full control over styling while maintaining accessibility and behavior.

**Key characteristics:**
- **Headless**: No styling included - bring your own styles (Tailwind, CSS, etc.)
- **Accessible**: Built-in ARIA attributes and keyboard navigation
- **Composable**: Build complex components from simple primitives
- **Type-safe**: Full TypeScript support

### When to Use Bits UI

Use Bits UI when building:
- **Interactive components**: Dropdowns, dialogs, popovers, tooltips
- **Form controls**: Select menus, comboboxes, radio groups, checkboxes
- **Navigation**: Tabs, accordions, collapsible sections
- **Data display**: Tables, pagination, calendars
- **Complex UI patterns**: Command palettes, context menus, sliders

### Accessing Documentation

**IMPORTANT**: Bits UI documentation follows the [llms.txt standard](https://llmstxt.org/), making it optimized for LLMs to parse and understand.

#### Fetch Documentation On Demand

When a user needs Bits UI components, fetch the relevant documentation using the `fetch` or `agentic_fetch` tool:

**For specific components:**
```
https://bits-ui.com/docs/components/[component-name]/llms.txt
```

**For complete documentation:**
```
https://bits-ui.com/docs/llms.txt
```

**For documentation index:**
```
https://bits-ui.com/llms.txt
```

#### Common Components

| Component | URL | Use Case |
|-----------|-----|----------|
| Accordion | `https://bits-ui.com/docs/components/accordion/llms.txt` | Collapsible sections |
| Button | `https://bits-ui.com/docs/components/button/llms.txt` | Accessible buttons |
| Checkbox | `https://bits-ui.com/docs/components/checkbox/llms.txt` | Checkbox inputs |
| Combobox | `https://bits-ui.com/docs/components/combobox/llms.txt` | Searchable select |
| Dialog | `https://bits-ui.com/docs/components/dialog/llms.txt` | Modal dialogs |
| Dropdown Menu | `https://bits-ui.com/docs/components/dropdown-menu/llms.txt` | Context menus |
| Popover | `https://bits-ui.com/docs/components/popover/llms.txt` | Floating content |
| Select | `https://bits-ui.com/docs/components/select/llms.txt` | Select menus |
| Tabs | `https://bits-ui.com/docs/components/tabs/llms.txt` | Tab navigation |
| Tooltip | `https://bits-ui.com/docs/components/tooltip/llms.txt` | Hover tooltips |

See full list at: https://bits-ui.com/llms.txt

### Workflow: Using Bits UI

1. **Identify component need**
   - User asks for dropdown, dialog, etc.

2. **Fetch relevant documentation**
   ```typescript
   // Use fetch or agentic_fetch tool
   fetch("https://bits-ui.com/docs/components/dialog/llms.txt", "text")
   ```

3. **Read the documentation**
   - Understand component API
   - Note required props and structure
   - Review accessibility features

4. **Implement component**
   - Use Bits UI primitives as building blocks
   - Add custom styling (Tailwind, CSS, etc.)
   - Follow Svelte 5 runes patterns

5. **Validate with autofixer**
   - Run `mcp_svelte_svelte-autofixer`
   - Apply any suggestions

### Example: Building a Dialog

```svelte
<script lang="ts">
  import { Dialog } from 'bits-ui';
  
  let open = $state(false);
</script>

<Dialog.Root bind:open>
  <Dialog.Trigger class="btn">Open Dialog</Dialog.Trigger>
  <Dialog.Portal>
    <Dialog.Overlay class="fixed inset-0 bg-black/50" />
    <Dialog.Content class="fixed left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2 bg-white p-6 rounded-lg">
      <Dialog.Title class="text-xl font-bold">Dialog Title</Dialog.Title>
      <Dialog.Description class="text-gray-600 mt-2">
        Dialog description goes here.
      </Dialog.Description>
      <Dialog.Close class="btn mt-4">Close</Dialog.Close>
    </Dialog.Content>
  </Dialog.Portal>
</Dialog.Root>
```

### Integration with Convex

Bits UI components work seamlessly with Convex data:

```svelte
<script lang="ts">
  import { useQuery, useConvexClient } from 'convex-svelte';
  import { Select } from 'bits-ui';
  import { api } from '$lib/convex/_generated/api.js';
  
  const client = useConvexClient();
  const users = useQuery(api.users.list, () => ({}));
  
  let selectedUserId = $state<string>('');
  
  async function assignUser() {
    await client.mutation(api.tasks.assign, {
      userId: selectedUserId
    });
  }
</script>

<Select.Root bind:value={selectedUserId}>
  <Select.Trigger>
    <Select.Value placeholder="Select user..." />
  </Select.Trigger>
  <Select.Content>
    {#if users.isLoading}
      <Select.Item value="" disabled>Loading...</Select.Item>
    {:else if users.data}
      {#each users.data as user}
        <Select.Item value={user._id}>{user.name}</Select.Item>
      {/each}
    {/if}
  </Select.Content>
</Select.Root>

<button onclick={assignUser}>Assign</button>
```

### Best Practices

1. **Fetch docs when needed** - Don't assume component API, always fetch latest docs
2. **Follow accessibility patterns** - Bits UI handles ARIA, don't override unnecessarily  
3. **Style consistently** - Use same styling approach across all components (Tailwind, CSS modules, etc.)
4. **Leverage composition** - Combine Bits UI primitives to build complex patterns
5. **Test interactions** - Verify keyboard navigation and screen reader support

### Notes

- Not all Bits UI pages support `/llms.txt` (e.g., Figma page is excluded)
- Check https://bits-ui.com/llms.txt for up-to-date list of available endpoints
- The "Copy Markdown" button on each doc page provides same content as `/llms.txt`
- Always prefer fetching docs over guessing component APIs

---

## Common Workflows

### Workflow 1: Build Svelte + Convex Feature

1. **Understand requirements**
   - Call `mcp_svelte_list-sections` to find relevant Svelte docs
   - Call `mcp_svelte_get-documentation` with all relevant sections
   - If building UI components, fetch Bits UI docs for needed components

2. **Inspect Convex deployment**
   - Call `mcp_convex_status` to get deployment selector
   - Call `mcp_convex_tables` to see schema
   - Call `mcp_convex_functionSpec` to see available functions

3. **Write backend (if needed)**
   - Create/update Convex functions in `convex/` folder
   - Use proper validators and types
   - Follow patterns in this guide

4. **Write frontend**
   - Use `convex-svelte` patterns
   - Use `useQuery` for data
   - Use `useConvexClient()` for mutations/actions
   - Use Bits UI for interactive components (dialogs, dropdowns, etc.)
   - Follow Svelte 5 runes ($state, $derived, etc.)

5. **Validate Svelte code**
   - Call `mcp_svelte_svelte-autofixer`
   - Apply fixes
   - Repeat until clean

6. **Test**
   - Use `mcp_convex_run` to test functions
   - Use `mcp_convex_logs` to debug
   - Test frontend in browser

### Workflow 2: Debug Convex Issue

1. **Get deployment info**
   - Call `mcp_convex_status`
   - Call `mcp_convex_tables` to check schema

2. **Inspect data**
   - Call `mcp_convex_data` to view table contents
   - Call `mcp_convex_runOneoffQuery` for custom queries

3. **Check function execution**
   - Call `mcp_convex_logs` to see recent executions
   - Look for errors or unexpected output

4. **Test functions**
   - Call `mcp_convex_run` to manually invoke functions
   - Verify behavior

5. **Fix and verify**
   - Update Convex functions
   - Re-test with MCP tools
   - Check logs again

### Workflow 3: Add New Feature to Existing App

1. **Understand existing code**
   - Call `mcp_convex_status` and `mcp_convex_functionSpec`
   - Review existing functions and schema

2. **Get Svelte docs**
   - Call `mcp_svelte_list-sections`
   - Call `mcp_svelte_get-documentation` for relevant sections

3. **Extend backend**
   - Add new Convex functions
   - Update schema if needed
   - Follow existing patterns

4. **Extend frontend**
   - Create new Svelte components
   - Use existing `convex-svelte` setup
   - Integrate with new backend functions

5. **Validate and test**
   - Run `mcp_svelte_svelte-autofixer` on new components
   - Test with `mcp_convex_run` and `mcp_convex_logs`

---

## Best Practices

### Convex Backend

1. **Always define validators** for args and returns
2. **Use indexes** instead of filters in queries
3. **Keep actions thin** - minimize calls to queries/mutations
4. **Use internal functions** for private logic
5. **Include proper error handling**
6. **Use TypeScript types** from `_generated/dataModel`
7. **Organize files thoughtfully** - use file-based routing effectively

### Convex + Svelte Frontend

1. **Call setupConvex once** in root layout
2. **Use useQuery for reactive data** - it auto-updates
3. **Skip queries conditionally** - return `'skip'` instead of wrapping in $derived
4. **Use SSR with initialData** to avoid loading states
5. **Keep mutations simple** - call `client.mutation()` directly
6. **Handle loading and error states** in UI

### Svelte 5 Patterns

1. **Use runes** - `$state`, `$derived`, `$effect`, `$props`
2. **Avoid legacy reactivity** - no `$:` in new Svelte 5 code
3. **Use snippets** instead of slots where appropriate
4. **Prefer fine-grained reactivity** over stores
5. **Validate with autofixer** before finalizing code

### Bits UI Patterns

1. **Fetch documentation first** - Always get latest component API from bits-ui.com/llms.txt
2. **Use composition** - Combine Root, Trigger, Content, etc. primitives
3. **Bind state reactively** - Use `bind:open`, `bind:value` with Svelte 5 runes
4. **Style with utility classes** - Tailwind works great with Bits UI
5. **Preserve accessibility** - Don't override ARIA attributes unless necessary

### MCP Tool Usage

1. **Always prefer MCP tools** over CLI commands
2. **Call status first** before any other Convex MCP tool
3. **Use logs liberally** for debugging
4. **Batch documentation requests** - fetch multiple sections at once
5. **Run autofixer until clean** - don't stop after one pass

---

## Troubleshooting

### effect_in_teardown Error

**Symptom:** Error when using `useQuery` in conditional components (modals, dialogs).

**Cause:** Wrapping `useQuery` in `$derived` block.

**Solution:** Use conditional skipping instead:

```svelte
<script lang="ts">
  // ❌ BAD - causes effect_in_teardown
  const query = $derived(isOpen ? useQuery(...) : null);

  // ✅ GOOD - use skip pattern
  const query = useQuery(
    api.data.get,
    () => isOpen ? { id } : 'skip'
  );
</script>
```

### Query Not Updating

**Symptom:** `useQuery` not reactively updating when arguments change.

**Solution:** Ensure arguments function returns new object:

```svelte
<script lang="ts">
  let filter = $state('');

  // ✅ GOOD - returns new object when filter changes
  const results = useQuery(
    api.search,
    () => ({ query: filter })
  );
</script>
```

### Convex Function Not Found

**Symptom:** "Function not found" error when calling via MCP.

**Solution:**
1. Check function is deployed: `mcp_convex_functionSpec`
2. Verify function path: `api.folder.file.functionName`
3. Check function visibility (public vs internal)

### Type Errors in Svelte Components

**Symptom:** TypeScript errors in Svelte files.

**Solution:**
1. Run `bunx convex dev` to generate types
2. Import from `convex/_generated/api` and `convex/_generated/dataModel`
3. Use `svelte-autofixer` to catch issues

---

## Quick Reference

### Convex Client Methods

```typescript
const client = useConvexClient();

// Query (reactive)
const data = useQuery(api.func, () => args);

// Mutation (imperative)
await client.mutation(api.func, args);

// Action (imperative)  
await client.action(api.func, args);
```

### Convex Function Types

| Type | Used For | Has ctx.db | Called From |
|------|----------|-----------|-------------|
| query | Read data | ✅ Yes | Client, queries, mutations, actions |
| mutation | Write data | ✅ Yes | Client, mutations, actions |
| action | External APIs | ❌ No | Client, actions |

### Validator Quick Reference

```typescript
v.null()           // null
v.number()         // number (Float64)
v.int64()          // bigint
v.string()         // string
v.boolean()        // boolean
v.bytes()          // ArrayBuffer
v.id("table")      // Id<"table">
v.array(v.string()) // string[]
v.object({ ... })  // { ... }
v.record(k, v)     // Record<k, v>
v.union(a, b)      // a | b
v.optional(v.string()) // string | undefined
v.literal("value") // "value"
```

### Convex Types Table

| Convex Type | TS/JS Type | Validator | Notes |
|------------|------------|-----------|-------|
| Id | string | `v.id(tableName)` | Document identifier |
| Null | null | `v.null()` | Use instead of undefined |
| Int64 | bigint | `v.int64()` | Between -2^63 and 2^63-1 |
| Float64 | number | `v.number()` | IEEE-754 double precision |
| Boolean | boolean | `v.boolean()` | |
| String | string | `v.string()` | UTF-8, must be valid Unicode |
| Bytes | ArrayBuffer | `v.bytes()` | First-class bytestrings |
| Array | Array | `v.array(values)` | Max 8192 values |
| Object | Object | `v.object({...})` | Plain objects only, max 1024 entries |
| Record | Record | `v.record(k, v)` | Dynamic keys at runtime |

### MCP Tool Call Order

**Starting new Convex + Svelte project:**
1. `mcp_svelte_list-sections`
2. `mcp_svelte_get-documentation` 
3. `mcp_convex_status`
4. Write backend and frontend
5. `mcp_svelte_svelte-autofixer`
6. `mcp_convex_run` (test)
7. `mcp_convex_logs` (debug)

**Debugging existing project:**
1. `mcp_convex_status`
2. `mcp_convex_logs`
3. `mcp_convex_functionSpec`
4. `mcp_convex_data` or `mcp_convex_runOneoffQuery`
5. Fix issues
6. `mcp_convex_run` (verify)

---

## Summary

- **Use Convex MCP tools** for all deployment interactions
- **Follow convex-svelte patterns** for frontend integration
- **Always validate Svelte code** with autofixer before sending to user
- **Prefer MCP over CLI** - use status, logs, run, etc. instead of command-line tools
- **Default to dev deployment** unless debugging production
- **Fetch all relevant docs at once** when using Svelte MCP
- **Use Svelte 5 runes** - $state, $derived, $effect, $props
- **Define validators for all Convex functions** - both args and returns
- **Use indexes in queries** - avoid filters for performance
- **Fetch Bits UI docs on demand** - don't assume component APIs, always get latest from bits-ui.com/llms.txt

This is your complete reference for working with Convex + Svelte + Bits UI projects.
