---
name: google-workspace
description: Add Google Workspace integration with granular permissions. Supports Gmail, Calendar, Drive, Docs, Sheets with per-channel permission controls. Owners can grant/revoke specific capabilities to different groups.
---

# Google Workspace Integration

This skill adds comprehensive Google Workspace capabilities with a permission system that controls which groups can access which services and operations.

## Features

- **Gmail**: Read, search, send emails, manage labels
- **Google Calendar**: Read events, create/update/delete events
- **Google Drive**: List files, read files, upload files, share files
- **Google Docs/Sheets**: Read and write documents
- **Permission System**: Granular control over what each channel can do
- **Owner Controls**: Only the owner can grant permissions and perform write operations
- **Read/Write Separation**: Channels can have read-only or read-write access

## Initial Questions

Ask the user:

> I'll set up Google Workspace integration with permission controls. First, a few questions:
>
> **1. Which Google services do you want to enable?**
> - Gmail (email)
> - Google Calendar (events)
> - Google Drive (files)
> - All of the above
>
> **2. What's your primary use case?**
> - Personal assistant (I'm the only user)
> - Shared with family/friends (multiple users with different access levels)
> - Work/team environment (strict permission controls needed)
>
> **3. Default permissions for new groups:**
> - Read-only access to everything (safe default)
> - No access by default (grant per-group)
> - Full access (for personal use only)

Store their answers and proceed to setup.

---

## Prerequisites

### 1. Check Existing Google Setup

```bash
ls -la ~/.google-workspace-mcp/ 2>/dev/null || echo "No Google Workspace config found"
```

If credentials exist, ask if they want to reuse them or create new ones.

### 2. Create Config Directory

```bash
mkdir -p ~/.google-workspace-mcp
```

### 3. GCP Project Setup

**USER ACTION REQUIRED**

Tell the user:

> I need you to set up Google Cloud OAuth credentials. This is similar to the Gmail setup, but with extended permissions.
>
> 1. Open https://console.cloud.google.com in your browser
> 2. Create a new project (or select existing) - click the project dropdown at the top
> 3. Name it something like "NanoClaw Workspace"

Wait for confirmation, then:

> 4. Now enable the required APIs. Go to **APIs & Services → Library** for each:
>    - **Gmail API** - Search "Gmail API", click Enable
>    - **Google Calendar API** - Search "Google Calendar API", click Enable
>    - **Google Drive API** - Search "Google Drive API", click Enable
>    - **Google Docs API** - Search "Google Docs API", click Enable
>    - **Google Sheets API** - Search "Google Sheets API", click Enable
>
> Let me know when all 5 APIs are enabled.

Wait for confirmation, then:

> 5. Now create OAuth credentials:
>    - Go to **APIs & Services → Credentials** (left sidebar)
>    - Click **+ CREATE CREDENTIALS** at the top
>    - Select **OAuth client ID**
>    - If prompted for consent screen:
>      - Choose "External"
>      - Fill in app name (e.g., "NanoClaw Workspace")
>      - Add your email as developer
>      - Add these scopes in "Scopes" step:
>        - `https://www.googleapis.com/auth/gmail.modify`
>        - `https://www.googleapis.com/auth/calendar`
>        - `https://www.googleapis.com/auth/drive`
>        - `https://www.googleapis.com/auth/documents`
>        - `https://www.googleapis.com/auth/spreadsheets`
>      - Save and continue
>    - For Application type, select **Desktop app**
>    - Name it "NanoClaw"
>    - Click **Create**

Wait for confirmation, then:

> 6. Download the credentials:
>    - Click **DOWNLOAD JSON** on the popup (or find it in the list and click download)
>    - Save it as `gcp-oauth.keys.json`
>
> Paste the full path to the file, or paste the JSON contents directly.

If user provides a path:

```bash
cp "/path/user/provided/gcp-oauth.keys.json" ~/.google-workspace-mcp/gcp-oauth.keys.json
```

If user pastes JSON:

```bash
cat > ~/.google-workspace-mcp/gcp-oauth.keys.json << 'EOF'
{paste JSON here}
EOF
```

Verify:

```bash
cat ~/.google-workspace-mcp/gcp-oauth.keys.json | head -10
```

### 4. Install Google Auth Library

```bash
npm install googleapis @google-cloud/local-auth --save
```

### 5. OAuth Authorization

**USER ACTION REQUIRED**

Tell the user:

> I'm going to create an authorization script. When you run it, a browser will open asking you to:
> 1. Sign in to Google
> 2. Grant access to Gmail, Calendar, Drive, Docs, Sheets
>
> **Important:** If you see "This app isn't verified", click "Advanced" then "Go to NanoClaw (unsafe)" - this is normal for personal OAuth apps.

Create the auth script:

```bash
cat > /tmp/google-workspace-auth.js << 'EOF'
const { authenticate } = require('@google-cloud/local-auth');
const { google } = require('googleapis');
const fs = require('fs');
const path = require('path');

const SCOPES = [
  'https://www.googleapis.com/auth/gmail.modify',
  'https://www.googleapis.com/auth/calendar',
  'https://www.googleapis.com/auth/drive',
  'https://www.googleapis.com/auth/documents',
  'https://www.googleapis.com/auth/spreadsheets',
];

const CONFIG_DIR = path.join(require('os').homedir(), '.google-workspace-mcp');
const KEYS_FILE = path.join(CONFIG_DIR, 'gcp-oauth.keys.json');
const TOKEN_FILE = path.join(CONFIG_DIR, 'credentials.json');

async function authorize() {
  const auth = await authenticate({
    scopes: SCOPES,
    keyfilePath: KEYS_FILE,
  });

  // Save credentials
  const credentials = {
    type: 'authorized_user',
    client_id: auth.credentials.client_id,
    client_secret: auth.credentials.client_secret,
    refresh_token: auth.credentials.refresh_token,
  };

  fs.writeFileSync(TOKEN_FILE, JSON.stringify(credentials, null, 2));
  console.log('Authorization successful! Credentials saved to:', TOKEN_FILE);
}

authorize().catch(console.error);
EOF
```

Run the auth script:

```bash
node /tmp/google-workspace-auth.js
```

Tell the user:

> Complete the authorization in your browser. Let me know when you've authorized.

Wait for confirmation, then verify:

```bash
if [ -f ~/.google-workspace-mcp/credentials.json ]; then
  echo "✓ Google Workspace authorization successful!"
  ls -la ~/.google-workspace-mcp/
else
  echo "✗ ERROR: credentials.json not found - authorization may have failed"
fi
```

Clean up:

```bash
rm -f /tmp/google-workspace-auth.js
```

---

## Implementation

### Step 1: Add Permissions Database Schema

Read `src/db.ts` and add the permissions table to the `createSchema` function:

```typescript
CREATE TABLE IF NOT EXISTS workspace_permissions (
  group_folder TEXT NOT NULL,
  service TEXT NOT NULL,
  permission TEXT NOT NULL,
  granted_at TEXT NOT NULL,
  granted_by TEXT,
  PRIMARY KEY (group_folder, service, permission)
);

CREATE INDEX IF NOT EXISTS idx_permissions_group ON workspace_permissions(group_folder);
```

Then add these helper functions to `src/db.ts`:

```typescript
export interface WorkspacePermission {
  group_folder: string;
  service: string;
  permission: string;
  granted_at: string;
  granted_by: string | null;
}

export function grantWorkspacePermission(
  groupFolder: string,
  service: string,
  permission: string,
  grantedBy?: string
): void {
  db.prepare(
    `INSERT OR REPLACE INTO workspace_permissions (group_folder, service, permission, granted_at, granted_by)
     VALUES (?, ?, ?, ?, ?)`
  ).run(groupFolder, service, permission, new Date().toISOString(), grantedBy || null);
}

export function revokeWorkspacePermission(
  groupFolder: string,
  service: string,
  permission: string
): void {
  db.prepare(
    `DELETE FROM workspace_permissions WHERE group_folder = ? AND service = ? AND permission = ?`
  ).run(groupFolder, service, permission);
}

export function hasWorkspacePermission(
  groupFolder: string,
  service: string,
  permission: string
): boolean {
  const row = db.prepare(
    `SELECT 1 FROM workspace_permissions WHERE group_folder = ? AND service = ? AND permission = ?`
  ).get(groupFolder, service, permission);
  return !!row;
}

export function getWorkspacePermissions(groupFolder: string): WorkspacePermission[] {
  return db.prepare(
    `SELECT * FROM workspace_permissions WHERE group_folder = ? ORDER BY service, permission`
  ).all(groupFolder) as WorkspacePermission[];
}

export function getAllWorkspacePermissions(): WorkspacePermission[] {
  return db.prepare(
    `SELECT * FROM workspace_permissions ORDER BY group_folder, service, permission`
  ).all() as WorkspacePermission[];
}

export function revokeAllPermissions(groupFolder: string): void {
  db.prepare('DELETE FROM workspace_permissions WHERE group_folder = ?').run(groupFolder);
}
```

### Step 2: Add Owner Configuration

Read `src/config.ts` and add owner configuration:

```typescript
// Owner JID - only this user can grant permissions and perform write operations
// Set this to your WhatsApp JID (find it in available_groups.json or main group logs)
export const OWNER_JID = process.env.OWNER_JID || '';

// Workspace default permissions for new groups
export const WORKSPACE_DEFAULT_PERMISSIONS: Array<{service: string; permission: string}> =
  JSON.parse(process.env.WORKSPACE_DEFAULT_PERMISSIONS || '[]');
```

### Step 3: Create Workspace Permission MCP Server

Create `container/workspace-permission-mcp/package.json`:

```bash
mkdir -p container/workspace-permission-mcp
cat > container/workspace-permission-mcp/package.json << 'EOF'
{
  "name": "workspace-permission-mcp",
  "version": "1.0.0",
  "type": "module",
  "main": "index.js",
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.4",
    "googleapis": "^144.0.0"
  }
}
EOF
```

Install dependencies:

```bash
cd container/workspace-permission-mcp && npm install && cd ../..
```

Create `container/workspace-permission-mcp/index.js`:

```javascript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { google } from 'googleapis';
import fs from 'fs';
import path from 'path';
import { z } from 'zod';

const homeDir = process.env.HOME || '/home/node';
const CREDENTIALS_PATH = path.join(homeDir, '.google-workspace-mcp', 'credentials.json');
const PERMISSIONS_FILE = process.env.WORKSPACE_PERMISSIONS_FILE || '/workspace/ipc/permissions.json';

// Read group context from environment
const groupFolder = process.env.NANOCLAW_GROUP_FOLDER;
const isOwner = process.env.NANOCLAW_IS_OWNER === '1';

// Load saved permissions (written by host)
function loadPermissions() {
  try {
    if (fs.existsSync(PERMISSIONS_FILE)) {
      return JSON.parse(fs.readFileSync(PERMISSIONS_FILE, 'utf-8'));
    }
  } catch (err) {
    console.error('[workspace-mcp] Failed to load permissions:', err);
  }
  return [];
}

function hasPermission(service, permission) {
  const permissions = loadPermissions();
  return permissions.some(p =>
    p.group_folder === groupFolder &&
    p.service === service &&
    p.permission === permission
  );
}

// Create OAuth2 client
async function getAuthClient() {
  const credentials = JSON.parse(fs.readFileSync(CREDENTIALS_PATH, 'utf-8'));
  const oauth2Client = new google.auth.OAuth2(
    credentials.client_id,
    credentials.client_secret,
    'http://localhost'
  );
  oauth2Client.setCredentials({
    refresh_token: credentials.refresh_token
  });
  return oauth2Client;
}

const server = new McpServer({
  name: 'google-workspace',
  version: '1.0.0',
});

// Gmail Tools
server.tool(
  'gmail_search',
  'Search emails in Gmail. Requires read_email permission.',
  {
    query: z.string().describe('Gmail search query (e.g., "from:john is:unread")'),
    maxResults: z.number().optional().describe('Maximum results (default 10)'),
  },
  async (args) => {
    if (!hasPermission('gmail', 'read_email')) {
      return {
        content: [{ type: 'text', text: 'Permission denied: gmail.read_email not granted to this channel' }],
        isError: true
      };
    }

    const auth = await getAuthClient();
    const gmail = google.gmail({ version: 'v1', auth });

    const res = await gmail.users.messages.list({
      userId: 'me',
      q: args.query,
      maxResults: args.maxResults || 10,
    });

    return {
      content: [{ type: 'text', text: JSON.stringify(res.data, null, 2) }]
    };
  }
);

server.tool(
  'gmail_send',
  'Send an email via Gmail. Requires send_email permission. Owner only.',
  {
    to: z.string().describe('Recipient email address'),
    subject: z.string().describe('Email subject'),
    body: z.string().describe('Email body (plain text)'),
  },
  async (args) => {
    if (!isOwner) {
      return {
        content: [{ type: 'text', text: 'Permission denied: Only the owner can send emails' }],
        isError: true
      };
    }

    if (!hasPermission('gmail', 'send_email')) {
      return {
        content: [{ type: 'text', text: 'Permission denied: gmail.send_email not granted to this channel' }],
        isError: true
      };
    }

    const auth = await getAuthClient();
    const gmail = google.gmail({ version: 'v1', auth });

    const message = [
      `To: ${args.to}`,
      `Subject: ${args.subject}`,
      '',
      args.body
    ].join('\n');

    const encoded = Buffer.from(message).toString('base64').replace(/\+/g, '-').replace(/\//g, '_').replace(/=+$/, '');

    const res = await gmail.users.messages.send({
      userId: 'me',
      requestBody: {
        raw: encoded
      }
    });

    return {
      content: [{ type: 'text', text: `Email sent successfully (ID: ${res.data.id})` }]
    };
  }
);

// Calendar Tools
server.tool(
  'calendar_list_events',
  'List upcoming calendar events. Requires read_calendar permission.',
  {
    maxResults: z.number().optional().describe('Maximum events to return (default 10)'),
    timeMin: z.string().optional().describe('Start time (ISO 8601, default now)'),
  },
  async (args) => {
    if (!hasPermission('calendar', 'read_calendar')) {
      return {
        content: [{ type: 'text', text: 'Permission denied: calendar.read_calendar not granted' }],
        isError: true
      };
    }

    const auth = await getAuthClient();
    const calendar = google.calendar({ version: 'v3', auth });

    const res = await calendar.events.list({
      calendarId: 'primary',
      timeMin: args.timeMin || new Date().toISOString(),
      maxResults: args.maxResults || 10,
      singleEvents: true,
      orderBy: 'startTime',
    });

    return {
      content: [{ type: 'text', text: JSON.stringify(res.data.items, null, 2) }]
    };
  }
);

server.tool(
  'calendar_create_event',
  'Create a calendar event. Requires create_event permission. Owner only.',
  {
    summary: z.string().describe('Event title'),
    start: z.string().describe('Start time (ISO 8601)'),
    end: z.string().describe('End time (ISO 8601)'),
    description: z.string().optional().describe('Event description'),
  },
  async (args) => {
    if (!isOwner) {
      return {
        content: [{ type: 'text', text: 'Permission denied: Only the owner can create events' }],
        isError: true
      };
    }

    if (!hasPermission('calendar', 'create_event')) {
      return {
        content: [{ type: 'text', text: 'Permission denied: calendar.create_event not granted' }],
        isError: true
      };
    }

    const auth = await getAuthClient();
    const calendar = google.calendar({ version: 'v3', auth });

    const event = {
      summary: args.summary,
      description: args.description,
      start: { dateTime: args.start },
      end: { dateTime: args.end },
    };

    const res = await calendar.events.insert({
      calendarId: 'primary',
      requestBody: event,
    });

    return {
      content: [{ type: 'text', text: `Event created: ${res.data.htmlLink}` }]
    };
  }
);

// Drive Tools
server.tool(
  'drive_list_files',
  'List files in Google Drive. Requires read_files permission.',
  {
    query: z.string().optional().describe('Search query (e.g., "name contains \'report\'")'),
    maxResults: z.number().optional().describe('Max results (default 10)'),
  },
  async (args) => {
    if (!hasPermission('drive', 'read_files')) {
      return {
        content: [{ type: 'text', text: 'Permission denied: drive.read_files not granted' }],
        isError: true
      };
    }

    const auth = await getAuthClient();
    const drive = google.drive({ version: 'v3', auth });

    const res = await drive.files.list({
      pageSize: args.maxResults || 10,
      q: args.query,
      fields: 'files(id, name, mimeType, modifiedTime, size)',
    });

    return {
      content: [{ type: 'text', text: JSON.stringify(res.data.files, null, 2) }]
    };
  }
);

server.tool(
  'drive_read_file',
  'Read a file from Google Drive. Requires read_files permission.',
  {
    fileId: z.string().describe('The Google Drive file ID'),
  },
  async (args) => {
    if (!hasPermission('drive', 'read_files')) {
      return {
        content: [{ type: 'text', text: 'Permission denied: drive.read_files not granted' }],
        isError: true
      };
    }

    const auth = await getAuthClient();
    const drive = google.drive({ version: 'v3', auth });

    const res = await drive.files.get({
      fileId: args.fileId,
      alt: 'media',
    }, { responseType: 'text' });

    return {
      content: [{ type: 'text', text: res.data }]
    };
  }
);

// Start server
const transport = new StdioServerTransport();
await server.connect(transport);
```

Build the workspace MCP:

```bash
cd container/workspace-permission-mcp && npm run build 2>/dev/null || echo "No build script needed for pure JS"
cd ../..
```

### Step 4: Mount Credentials in Container

Read `src/container-runner.ts` and add the Google Workspace credentials mount (similar to Gmail):

```typescript
// Google Workspace credentials directory
const workspaceDir = path.join(homeDir, '.google-workspace-mcp');
if (fs.existsSync(workspaceDir)) {
  mounts.push({
    hostPath: workspaceDir,
    containerPath: '/home/node/.google-workspace-mcp',
    readonly: false  // May need to refresh tokens
  });
}

// Write current permissions to a file the MCP can read
const ipcDir = path.join(sessionDir, 'ipc');
const permissionsFile = path.join(ipcDir, 'permissions.json');
fs.mkdirSync(ipcDir, { recursive: true });

// Import permission functions
import { getWorkspacePermissions } from './db.js';
const permissions = getWorkspacePermissions(groupFolder);
fs.writeFileSync(permissionsFile, JSON.stringify(permissions, null, 2));
```

### Step 5: Add Workspace MCP to Agent Runner

Read `container/agent-runner/src/index.ts` and add the workspace MCP to the `mcpServers` config:

```typescript
// In the query() call, add to mcpServers:
workspace: {
  command: 'node',
  args: ['/app/workspace-permission-mcp/index.js'],
  env: {
    NANOCLAW_GROUP_FOLDER: containerInput.groupFolder,
    NANOCLAW_IS_OWNER: containerInput.isOwner ? '1' : '0',
    WORKSPACE_PERMISSIONS_FILE: '/workspace/ipc/permissions.json',
  }
}

// In allowedTools, add:
'mcp__workspace__*'
```

The result should look like:

```typescript
mcpServers: {
  nanoclaw: ipcMcp,
  workspace: {
    command: 'node',
    args: ['/app/workspace-permission-mcp/index.js'],
    env: {
      NANOCLAW_GROUP_FOLDER: containerInput.groupFolder,
      NANOCLAW_IS_OWNER: containerInput.isOwner ? '1' : '0',
      WORKSPACE_PERMISSIONS_FILE: '/workspace/ipc/permissions.json',
    }
  }
},
allowedTools: [
  'Bash',
  'Read', 'Write', 'Edit', 'Glob', 'Grep',
  'WebSearch', 'WebFetch',
  'mcp__nanoclaw__*',
  'mcp__workspace__*'
],
```

### Step 6: Add Owner Detection to Container Input

Read `container/agent-runner/src/index.ts` and add `isOwner` to the `ContainerInput` interface:

```typescript
interface ContainerInput {
  prompt: string;
  sessionId?: string;
  groupFolder: string;
  chatJid: string;
  isMain: boolean;
  isScheduledTask?: boolean;
  isOwner?: boolean;
  secrets?: Record<string, string>;
}
```

Read `src/container-runner.ts` and find where `ContainerInput` is built, then add:

```typescript
import { OWNER_JID } from './config.js';

// In the containerInput object:
isOwner: chatJid === OWNER_JID || isMain,
```

### Step 7: Add Permission Management Tools to IPC MCP

Read `container/agent-runner/src/ipc-mcp-stdio.ts` and add permission management tools:

```typescript
server.tool(
  'workspace_grant_permission',
  'Grant a Google Workspace permission to a group. Main/owner only. Available services: gmail, calendar, drive, docs, sheets. Available permissions: read_email, send_email, read_calendar, create_event, read_files, write_files, read_docs, create_docs, read_sheets, create_sheets, update_sheets.',
  {
    group_folder: z.string().describe('Group folder name (e.g., "family-chat")'),
    service: z.enum(['gmail', 'calendar', 'drive', 'docs', 'sheets']).describe('Service to grant access to'),
    permission: z.string().describe('Permission to grant (e.g., "read_email", "send_email", "read_docs", "read_sheets")'),
  },
  async (args) => {
    if (!isMain) {
      return {
        content: [{ type: 'text', text: 'Permission denied: Only main/owner can grant permissions' }],
        isError: true
      };
    }

    const data = {
      type: 'grant_permission',
      groupFolder: args.group_folder,
      service: args.service,
      permission: args.permission,
      grantedBy: groupFolder,
      timestamp: new Date().toISOString(),
    };

    writeIpcFile(TASKS_DIR, data);

    return {
      content: [{ type: 'text', text: `Permission ${args.service}.${args.permission} granted to ${args.group_folder}` }]
    };
  }
);

server.tool(
  'workspace_revoke_permission',
  'Revoke a Google Workspace permission from a group. Main/owner only.',
  {
    group_folder: z.string().describe('Group folder name'),
    service: z.enum(['gmail', 'calendar', 'drive', 'docs', 'sheets']).describe('Service'),
    permission: z.string().describe('Permission to revoke'),
  },
  async (args) => {
    if (!isMain) {
      return {
        content: [{ type: 'text', text: 'Permission denied: Only main/owner can revoke permissions' }],
        isError: true
      };
    }

    const data = {
      type: 'revoke_permission',
      groupFolder: args.group_folder,
      service: args.service,
      permission: args.permission,
      timestamp: new Date().toISOString(),
    };

    writeIpcFile(TASKS_DIR, data);

    return {
      content: [{ type: 'text', text: `Permission ${args.service}.${args.permission} revoked from ${args.group_folder}` }]
    };
  }
);

server.tool(
  'workspace_list_permissions',
  'List Google Workspace permissions for a group (or all groups if main).',
  {
    group_folder: z.string().optional().describe('Specific group to check (main only, omit to see all)'),
  },
  async (args) => {
    const permissionsFile = path.join(IPC_DIR, 'permissions.json');

    try {
      if (!fs.existsSync(permissionsFile)) {
        return { content: [{ type: 'text', text: 'No permissions configured yet.' }] };
      }

      const allPermissions = JSON.parse(fs.readFileSync(permissionsFile, 'utf-8'));

      let permissions;
      if (isMain && args.group_folder) {
        permissions = allPermissions.filter((p: { group_folder: string }) =>
          p.group_folder === args.group_folder
        );
      } else if (isMain) {
        permissions = allPermissions;
      } else {
        permissions = allPermissions.filter((p: { group_folder: string }) =>
          p.group_folder === groupFolder
        );
      }

      if (permissions.length === 0) {
        return { content: [{ type: 'text', text: 'No permissions found.' }] };
      }

      const formatted = permissions
        .map((p: { group_folder: string; service: string; permission: string; granted_at: string }) =>
          `- ${p.group_folder}: ${p.service}.${p.permission} (granted ${p.granted_at})`
        )
        .join('\n');

      return { content: [{ type: 'text', text: `Permissions:\n${formatted}` }] };
    } catch (err) {
      return {
        content: [{ type: 'text', text: `Error: ${err instanceof Error ? err.message : String(err)}` }]
      };
    }
  }
);

server.tool(
  'workspace_apply_template',
  'Apply a permission template to a group. Main/owner only. Available templates: personal-full, family-readonly, work-readonly, calendar-only, email-only, docs-editor.',
  {
    group_folder: z.string().describe('Group folder name to apply template to'),
    template: z.enum(['personal-full', 'family-readonly', 'work-readonly', 'calendar-only', 'email-only', 'docs-editor']).describe('Template name'),
    replace: z.boolean().optional().describe('If true, remove existing permissions before applying template (default: false)'),
  },
  async (args) => {
    if (!isMain) {
      return {
        content: [{ type: 'text', text: 'Permission denied: Only main/owner can apply templates' }],
        isError: true,
      };
    }

    const data = {
      type: 'apply_template',
      groupFolder: args.group_folder,
      template: args.template,
      replace: args.replace || false,
      grantedBy: groupFolder,
      timestamp: new Date().toISOString(),
    };

    writeIpcFile(TASKS_DIR, data);

    return {
      content: [{ type: 'text', text: `Template "${args.template}" ${args.replace ? 'applied (replaced existing)' : 'applied'} to ${args.group_folder}` }]
    };
  },
);

server.tool(
  'workspace_list_templates',
  'List available permission templates with descriptions.',
  {},
  async () => {
    const templates = {
      'personal-full': 'Full access to all Google Workspace services (personal use)',
      'family-readonly': 'Read-only access for family members (calendar, drive, docs, sheets)',
      'work-readonly': 'Read-only access to work-related services (email, calendar, drive, docs, sheets)',
      'calendar-only': 'Calendar access only',
      'email-only': 'Email read access only',
      'docs-editor': 'Read and edit Docs/Sheets (includes drive read)',
    };

    const formatted = Object.entries(templates)
      .map(([name, desc]) => `*${name}*\n  ${desc}`)
      .join('\n\n');

    return {
      content: [{ type: 'text', text: `Available templates:\n\n${formatted}` }]
    };
  },
);
```

### Step 8: Add Permission IPC Handlers

Read `src/ipc.ts` and add handlers for permission management:

```typescript
import {
  grantWorkspacePermission,
  revokeWorkspacePermission,
  getAllWorkspacePermissions
} from './db.js';

// In processTaskIpc function, add these cases:

case 'grant_permission': {
  const { groupFolder, service, permission, grantedBy } = parsed;
  grantWorkspacePermission(groupFolder, service, permission, grantedBy);

  // Update permissions file for running containers
  const permissionsFile = path.join(DATA_DIR, 'permissions.json');
  const allPerms = getAllWorkspacePermissions();
  fs.writeFileSync(permissionsFile, JSON.stringify(allPerms, null, 2));

  logger.info({ groupFolder, service, permission }, 'Permission granted');
  break;
}

case 'revoke_permission': {
  const { groupFolder, service, permission } = parsed;
  revokeWorkspacePermission(groupFolder, service, permission);

  // Update permissions file
  const permissionsFile = path.join(DATA_DIR, 'permissions.json');
  const allPerms = getAllWorkspacePermissions();
  fs.writeFileSync(permissionsFile, JSON.stringify(allPerms, null, 2));

  logger.info({ groupFolder, service, permission }, 'Permission revoked');
  break;
}

case 'apply_template': {
  const groupFolder = (data as any).groupFolder;
  const template = (data as any).template;
  const replace = (data as any).replace;
  const grantedBy = (data as any).grantedBy;

  if (groupFolder && template && WORKSPACE_PERMISSION_TEMPLATES[template]) {
    const templatePerms = WORKSPACE_PERMISSION_TEMPLATES[template].permissions;
    applyPermissionTemplate(groupFolder, templatePerms, grantedBy, replace);

    // Update permissions file
    const permissionsFile = path.join(DATA_DIR, 'permissions.json');
    const allPerms = getAllWorkspacePermissions();
    fs.writeFileSync(permissionsFile, JSON.stringify(allPerms, null, 2));

    logger.info({ groupFolder, template, replace }, 'Permission template applied');
  } else {
    logger.warn({ data }, 'Invalid apply_template request - missing required fields or unknown template');
  }
  break;
}
```

Also update the imports at the top of the file to include:

```typescript
import { WORKSPACE_PERMISSION_TEMPLATES } from './config.js';
import { applyPermissionTemplate } from './db.js';
```

### Step 9: Copy Workspace MCP into Container

Read `container/Dockerfile` and add the workspace MCP:

```dockerfile
# After the agent-runner COPY, add:
COPY workspace-permission-mcp /app/workspace-permission-mcp
RUN cd /app/workspace-permission-mcp && npm install --production
```

### Step 10: Update Group Memory

Update `groups/global/CLAUDE.md` to document the workspace tools:

```markdown

## Google Workspace

You have access to Google Workspace via permission-controlled tools:

### Gmail
- `mcp__workspace__gmail_search` - Search emails (requires gmail.read_email)
- `mcp__workspace__gmail_get` - Get full email content by ID (requires gmail.read_email)
- `mcp__workspace__gmail_send` - Send email (requires gmail.send_email, owner only)

### Calendar
- `mcp__workspace__calendar_list_events` - List upcoming events (requires calendar.read_calendar)
- `mcp__workspace__calendar_create_event` - Create event (requires calendar.create_event, owner only)

### Drive
- `mcp__workspace__drive_list_files` - List files (requires drive.read_files)
- `mcp__workspace__drive_read_file` - Read file content (requires drive.read_files)

### Docs
- `mcp__workspace__docs_read` - Read Google Doc content (requires docs.read_docs)
- `mcp__workspace__docs_create` - Create new Google Doc (requires docs.create_docs, owner only)

### Sheets
- `mcp__workspace__sheets_read` - Read Google Sheets data (requires sheets.read_sheets)
- `mcp__workspace__sheets_create` - Create new spreadsheet (requires sheets.create_sheets, owner only)
- `mcp__workspace__sheets_update` - Update spreadsheet data (requires sheets.update_sheets, owner only)

### Permission Management (Main/Owner Only)
- `mcp__nanoclaw__workspace_grant_permission` - Grant permission to a group
- `mcp__nanoclaw__workspace_revoke_permission` - Revoke permission from a group
- `mcp__nanoclaw__workspace_list_permissions` - List current permissions
- `mcp__nanoclaw__workspace_apply_template` - Apply permission template (e.g., "family-readonly", "work-readonly")
- `mcp__nanoclaw__workspace_list_templates` - List available templates

**Permission Templates:** Use `workspace_apply_template` to quickly set up common permission sets like "personal-full", "family-readonly", "work-readonly", "calendar-only", "email-only", or "docs-editor".

**Note:** Write operations (send email, create event, create/update docs/sheets) are restricted to the owner regardless of permissions.
```

Also update `groups/main/CLAUDE.md` with the same content.

### Step 11: Set Owner JID

Tell the user:

> I need to set your owner JID so the system knows who can perform write operations.
>
> Your WhatsApp JID should be in the format: `1234567890@s.whatsapp.net`
>
> I can find it by checking your recent messages. What's your phone number (with country code, e.g., +1234567890)?

When user provides their number, search for their JID:

```bash
sqlite3 store/messages.db "SELECT DISTINCT sender FROM messages WHERE sender LIKE '%$(echo "+1234567890" | sed 's/+//; s/@.*//')%' LIMIT 1"
```

Then set it in the environment:

```bash
# Add to .env file
echo "OWNER_JID=their_jid_here" >> .env
```

Or tell them to add it manually to their shell profile or LaunchDaemon plist.

### Step 12: Apply Default Permissions

Ask the user which default permissions they want for the main group based on their earlier answer:

```bash
# Example: Grant read-only access to main group
node -e "
const db = require('better-sqlite3')('store/messages.db');
const now = new Date().toISOString();

const permissions = [
  ['main', 'gmail', 'read_email'],
  ['main', 'calendar', 'read_calendar'],
  ['main', 'drive', 'read_files'],
];

for (const [group, service, perm] of permissions) {
  db.prepare(
    'INSERT OR REPLACE INTO workspace_permissions (group_folder, service, permission, granted_at, granted_by) VALUES (?, ?, ?, ?, ?)'
  ).run(group, service, perm, now, 'setup');
}

console.log('Default permissions granted to main group');
db.close();
"
```

### Step 13: Rebuild Container and Restart

Rebuild the container with workspace MCP:

```bash
cd container && ./build.sh
```

Wait for build, then compile TypeScript:

```bash
cd .. && npm run build
```

Restart the service:

```bash
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

Verify startup:

```bash
sleep 3 && tail -30 logs/nanoclaw.log
```

### Step 14: Test Integration

Tell the user:

> Google Workspace integration is ready! Test it by sending these messages in your main WhatsApp channel:
>
> **Check permissions:**
> `@Andy workspace_list_permissions`
>
> **Search Gmail (if read_email granted):**
> `@Andy gmail_search query:"is:unread"`
>
> **List calendar events (if read_calendar granted):**
> `@Andy calendar_list_events maxResults:5`
>
> **Grant permissions to another group (from main):**
> `@Andy workspace_grant_permission group_folder:"family-chat" service:"calendar" permission:"read_calendar"`

Monitor logs for any errors:

```bash
tail -f logs/nanoclaw.log
```

---

## Permission Reference

### Available Services
- `gmail` - Gmail access
- `calendar` - Google Calendar access
- `drive` - Google Drive access
- `docs` - Google Docs access
- `sheets` - Google Sheets access

### Available Permissions

#### Gmail
- `read_email` - Search and read emails
- `send_email` - Send emails (owner only)

#### Calendar
- `read_calendar` - List and read events
- `create_event` - Create new events (owner only)
- `update_event` - Modify existing events (owner only)
- `delete_event` - Delete events (owner only)

#### Drive
- `read_files` - List and read files
- `write_files` - Upload and modify files (owner only)

#### Docs
- `read_docs` - Read Google Docs content
- `create_docs` - Create new Google Docs (owner only)

#### Sheets
- `read_sheets` - Read Google Sheets data
- `create_sheets` - Create new spreadsheets (owner only)
- `update_sheets` - Update spreadsheet data (owner only)

---

## Permission Templates

Instead of granting individual permissions, you can use predefined templates:

### Available Templates

**personal-full**
- Full access to all Google Workspace services
- Includes read and write for all services
- Best for: Personal use with full control

**family-readonly**
- Read-only access for family members
- Calendar, Drive, Docs, Sheets (read-only)
- Best for: Family sharing without modification rights

**work-readonly**
- Read-only access to work services
- Email, Calendar, Drive, Docs, Sheets (read-only)
- Best for: Work groups with view-only access

**calendar-only**
- Calendar read access only
- Best for: Simple calendar sharing

**email-only**
- Email read access only
- Best for: Email monitoring groups

**docs-editor**
- Read and edit Docs/Sheets
- Includes Drive read access
- Best for: Document collaboration groups

### Apply a Template

```
@Andy workspace_apply_template group_folder:"family-chat" template:"family-readonly"
```

### Replace Existing Permissions with Template

```
@Andy workspace_apply_template group_folder:"work-team" template:"work-readonly" replace:true
```

### List Available Templates

```
@Andy workspace_list_templates
```

---

## Usage Examples

### Grant Read-Only Access to a Family Chat

```
@Andy workspace_grant_permission group_folder:"family-chat" service:"calendar" permission:"read_calendar"
```

### Grant Full Gmail Access to Main

```
@Andy workspace_grant_permission group_folder:"main" service:"gmail" permission:"read_email"
@Andy workspace_grant_permission group_folder:"main" service:"gmail" permission:"send_email"
```

### Revoke Permission

```
@Andy workspace_revoke_permission group_folder:"family-chat" service:"gmail" permission:"send_email"
```

### List All Permissions (from Main)

```
@Andy workspace_list_permissions
```

### List Permissions for Specific Group

```
@Andy workspace_list_permissions group_folder:"family-chat"
```

---

## Troubleshooting

### "Permission denied" errors

- Check permissions: `@Andy workspace_list_permissions`
- Verify the group has the required permission
- For write operations, ensure you're messaging from the owner JID

### OAuth token expired

```bash
# Re-authorize
rm ~/.google-workspace-mcp/credentials.json
node /tmp/google-workspace-auth.js  # (recreate the script from Step 5)
```

### Container can't access credentials

```bash
# Verify mount
cat groups/main/logs/container-*.log | grep "workspace-mcp"

# Check credentials exist
ls -la ~/.google-workspace-mcp/
```

### Permissions not updating

```bash
# Manually refresh permissions file
node -e "
const db = require('better-sqlite3')('store/messages.db');
const fs = require('fs');
const perms = db.prepare('SELECT * FROM workspace_permissions').all();
fs.writeFileSync('data/permissions.json', JSON.stringify(perms, null, 2));
console.log('Permissions file refreshed');
"
```

---

## Removing Google Workspace Integration

To completely remove the integration:

1. Remove from `container/agent-runner/src/index.ts`:
   - Delete `workspace` from `mcpServers`
   - Remove `mcp__workspace__*` from `allowedTools`

2. Remove from `src/container-runner.ts`:
   - Delete the `~/.google-workspace-mcp` mount block
   - Delete the permissions file write code

3. Remove from `container/agent-runner/src/ipc-mcp-stdio.ts`:
   - Delete all `workspace_*` tools

4. Remove from `src/ipc.ts`:
   - Delete `grant_permission` and `revoke_permission` cases

5. Remove workspace MCP directory:
   ```bash
   rm -rf container/workspace-permission-mcp
   ```

6. Remove from `container/Dockerfile`:
   - Delete COPY and RUN lines for workspace-permission-mcp

7. Drop database table:
   ```bash
   sqlite3 store/messages.db "DROP TABLE workspace_permissions"
   ```

8. Remove credentials:
   ```bash
   rm -rf ~/.google-workspace-mcp
   ```

9. Rebuild:
   ```bash
   cd container && ./build.sh && cd ..
   npm run build
   launchctl kickstart -k gui/$(id -u)/com.nanoclaw
   ```
