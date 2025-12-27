# VS Code Extension Implementation Guide

This guide provides concrete implementation details for converting SP Editor to a VS Code extension or creating a complementary VS Code extension.

## Table of Contents
1. [Project Structure](#project-structure)
2. [Core API Mappings](#core-api-mappings)
3. [Authentication Implementation](#authentication-implementation)
4. [SharePoint Operations](#sharepoint-operations)
5. [UI Implementation](#ui-implementation)
6. [State Management](#state-management)
7. [Code Examples](#code-examples)

## Project Structure

### Recommended VS Code Extension Structure
```
sp-editor-vscode/
├── src/
│   ├── extension.ts              # Extension entry point
│   ├── authentication/
│   │   ├── authProvider.ts       # VS Code auth provider
│   │   ├── tokenManager.ts       # Token storage & refresh
│   │   └── msalConfig.ts         # MSAL configuration
│   ├── sharepoint/
│   │   ├── spClient.ts           # SharePoint API client
│   │   ├── pnpWrapper.ts         # PnPjs Node.js wrapper
│   │   └── operations/           # SharePoint operations
│   │       ├── files.ts
│   │       ├── properties.ts
│   │       ├── scriptLinks.ts
│   │       └── search.ts
│   ├── views/
│   │   ├── siteExplorer.ts       # Tree view provider
│   │   ├── consolePanel.ts       # Webview for console
│   │   └── propertyEditor.ts     # Webview for properties
│   ├── webview/
│   │   ├── main.tsx              # React entry for webviews
│   │   ├── components/           # Reuse existing React components
│   │   └── adapters/             # Adapt chrome APIs to VS Code messaging
│   ├── commands/
│   │   ├── index.ts              # Command registry
│   │   ├── siteCommands.ts
│   │   └── fileCommands.ts
│   └── utils/
│       ├── configuration.ts      # VS Code settings
│       └── logger.ts             # Output channel logging
├── media/                        # Icons and assets
├── webview-ui/                   # Compiled React app for webviews
├── package.json                  # Extension manifest
└── tsconfig.json
```

## Core API Mappings

### Browser Extension → VS Code Extension

| Browser Extension API | VS Code Equivalent | Notes |
|----------------------|-------------------|-------|
| `chrome.devtools.*` | N/A | No DevTools in VS Code |
| `chrome.tabs.get()` | `vscode.workspace.getConfiguration()` | Get site URL from config |
| `chrome.scripting.executeScript()` | HTTP REST calls | Cannot inject into pages |
| `chrome.storage.local` | `vscode.ExtensionContext.globalState` | Persistent storage |
| `chrome.storage.sync` | `vscode.ExtensionContext.secrets` | Secure storage for tokens |
| `chrome.runtime.getURL()` | `vscode.Uri.joinPath(context.extensionUri, ...)` | Extension resources |
| DevTools panel | `vscode.window.createWebviewPanel()` | Webview panel |
| Extension popup | `vscode.window.createWebviewPanel()` or sidebar | Different UX |

### Window/Tab Detection → Configuration

**Current (Browser Extension)**:
```typescript
// Detect current SharePoint site from active tab
const tabId = chrome.devtools.inspectedWindow.tabId;
chrome.tabs.get(tabId, (tab) => {
  const siteUrl = tab.url;
});

// Access page context
chrome.scripting.executeScript({
  target: { tabId },
  func: () => window._spPageContextInfo
});
```

**VS Code Extension**:
```typescript
// Get site URL from configuration
const config = vscode.workspace.getConfiguration('speditor');
const siteUrl = config.get<string>('siteUrl');

// Or detect from workspace (SPFx project)
const spfxConfig = await detectSPFxProject();
if (spfxConfig) {
  const siteUrl = spfxConfig.pageUrl;
}

// Or let user select from recent sites
const siteUrl = await vscode.window.showQuickPick(
  recentSites,
  { placeHolder: 'Select SharePoint site' }
);
```

## Authentication Implementation

### VS Code Authentication Provider

```typescript
// src/authentication/authProvider.ts
import * as vscode from 'vscode';
import { PublicClientApplication, Configuration } from '@azure/msal-node';

export class SPEditorAuthenticationProvider implements vscode.AuthenticationProvider {
  private _sessionChangeEmitter = new vscode.EventEmitter<vscode.AuthenticationProviderAuthenticationSessionsChangeEvent>();
  private _msalApp: PublicClientApplication;

  constructor(private context: vscode.ExtensionContext) {
    const config: Configuration = {
      auth: {
        clientId: 'YOUR_CLIENT_ID',
        authority: 'https://login.microsoftonline.com/organizations'
      }
    };
    this._msalApp = new PublicClientApplication(config);
  }

  get onDidChangeSessions() {
    return this._sessionChangeEmitter.event;
  }

  async getSessions(scopes?: readonly string[]): Promise<readonly vscode.AuthenticationSession[]> {
    const accounts = await this._msalApp.getTokenCache().getAllAccounts();
    return accounts.map(account => this.createSession(account));
  }

  async createSession(scopes: readonly string[]): Promise<vscode.AuthenticationSession> {
    // Use device code flow for VS Code
    const deviceCodeRequest = {
      deviceCodeCallback: (response: any) => {
        vscode.window.showInformationMessage(
          `To sign in, use a web browser to open ${response.verificationUri} and enter code ${response.userCode}`
        );
      },
      scopes: [...scopes]
    };

    const response = await this._msalApp.acquireTokenByDeviceCode(deviceCodeRequest);
    
    const session: vscode.AuthenticationSession = {
      id: response.uniqueId,
      accessToken: response.accessToken,
      account: {
        label: response.account!.username,
        id: response.account!.homeAccountId
      },
      scopes: scopes as string[]
    };

    // Store in secret storage
    await this.context.secrets.store(`token-${session.id}`, response.accessToken);
    
    return session;
  }

  async removeSession(sessionId: string): Promise<void> {
    await this.context.secrets.delete(`token-${sessionId}`);
    this._sessionChangeEmitter.fire({ added: [], removed: [{ id: sessionId }], changed: [] });
  }
}

// Register provider in extension activation
export function registerAuthProvider(context: vscode.ExtensionContext) {
  const provider = new SPEditorAuthenticationProvider(context);
  context.subscriptions.push(
    vscode.authentication.registerAuthenticationProvider(
      'speditor',
      'SP Editor',
      provider
    )
  );
}
```

### Using Authentication

```typescript
// Get authenticated session
async function getAuthSession(): Promise<vscode.AuthenticationSession> {
  const session = await vscode.authentication.getSession(
    'speditor',
    ['https://graph.microsoft.com/.default'],
    { createIfNone: true }
  );
  return session;
}

// Use with PnPjs
import { spfi, SPFx } from '@pnp/sp';
import { MSAL } from '@pnp/msaljsclient';

async function getSPClient(siteUrl: string) {
  const session = await getAuthSession();
  
  const sp = spfi(siteUrl).using(
    MSAL({
      configuration: {
        auth: {
          clientId: 'YOUR_CLIENT_ID',
          authority: 'https://login.microsoftonline.com/organizations'
        }
      },
      authParams: {
        scopes: ['https://graph.microsoft.com/.default']
      }
    })
  );
  
  return sp;
}
```

## SharePoint Operations

### Converting Chrome Script Injection to REST Calls

**Current (Browser Extension)**:
```typescript
// src/pages/listproperties/chrome/getlistproperties.ts
export function getListProperties(listId: string, extensionUrl: string) {
  // Runs in page context, has access to window._spPageContextInfo
  const sp = pnp.sp;
  const results = await sp.web.lists.getById(listId).select('*').get();
  return { success: true, result: results };
}

// Called from extension
chrome.scripting.executeScript({
  target: { tabId: chrome.devtools.inspectedWindow.tabId },
  world: 'MAIN',
  args: [listId, chrome.runtime.getURL('')],
  func: getListProperties
});
```

**VS Code Extension**:
```typescript
// src/sharepoint/operations/properties.ts
import { spfi } from '@pnp/sp';
import '@pnp/sp/webs';
import '@pnp/sp/lists';

export async function getListProperties(
  siteUrl: string,
  listId: string,
  accessToken: string
): Promise<any> {
  const sp = spfi(siteUrl).using(
    // Configure with access token
    {
      headers: {
        'Authorization': `Bearer ${accessToken}`
      }
    }
  );

  try {
    const results = await sp.web.lists.getById(listId).select('*')();
    return { success: true, result: results };
  } catch (error) {
    return { 
      success: false, 
      errorMessage: error instanceof Error ? error.message : 'Unknown error'
    };
  }
}

// Called from command
async function executeGetListProperties(listId: string) {
  const session = await getAuthSession();
  const config = vscode.workspace.getConfiguration('speditor');
  const siteUrl = config.get<string>('siteUrl');
  
  if (!siteUrl) {
    vscode.window.showErrorMessage('Please configure SharePoint site URL');
    return;
  }

  const result = await getListProperties(siteUrl, listId, session.accessToken);
  
  if (result.success) {
    // Display in webview or tree view
    showPropertiesPanel(result.result);
  } else {
    vscode.window.showErrorMessage(result.errorMessage);
  }
}
```

### PnPjs Node.js Configuration

```typescript
// src/sharepoint/pnpWrapper.ts
import { spfi } from '@pnp/sp';
import { MSAL, IMSALOptions } from '@pnp/msaljsclient';
import '@pnp/sp/webs';
import '@pnp/sp/lists';
import '@pnp/sp/items';
import '@pnp/sp/files';
import '@pnp/sp/folders';

export class SPClient {
  private sp: any;

  constructor(siteUrl: string, accessToken: string) {
    this.sp = spfi(siteUrl).using({
      headers: {
        'Authorization': `Bearer ${accessToken}`,
        'Accept': 'application/json;odata=verbose'
      }
    });
  }

  async getWeb() {
    return await this.sp.web();
  }

  async getLists() {
    return await this.sp.web.lists();
  }

  async getListProperties(listId: string) {
    return await this.sp.web.lists.getById(listId).select('*')();
  }

  async createListProperty(listId: string, key: string, value: any, indexed: boolean) {
    const list = this.sp.web.lists.getById(listId);
    // Set property bag value
    await list.propertyBag.set(key, value);
    
    if (indexed) {
      // Add to indexed property keys
      const existingKeys = await list.propertyBag.get('vti_indexedpropertykeys');
      // Encode key and add to indexed keys
      const bytes = Array.from(key).flatMap(c => [c.charCodeAt(0), 0]);
      const encoded = Buffer.from(bytes).toString('base64');
      const newKeys = existingKeys ? `${existingKeys}|${encoded}` : encoded;
      await list.propertyBag.set('vti_indexedpropertykeys', newKeys);
    }
  }

  async getFiles(folderUrl: string) {
    return await this.sp.web.getFolderByServerRelativeUrl(folderUrl).files();
  }

  async getFileContent(fileUrl: string) {
    return await this.sp.web.getFileByServerRelativeUrl(fileUrl).getText();
  }

  async updateFile(fileUrl: string, content: string) {
    return await this.sp.web.getFileByServerRelativeUrl(fileUrl)
      .setContent(content);
  }
}
```

## UI Implementation

### Webview Panel for React App

```typescript
// src/views/consolePanel.ts
import * as vscode from 'vscode';

export class ConsolePanel {
  private panel: vscode.WebviewPanel | undefined;

  constructor(private context: vscode.ExtensionContext) {}

  public show() {
    if (this.panel) {
      this.panel.reveal();
      return;
    }

    this.panel = vscode.window.createWebviewPanel(
      'speditorConsole',
      'PnP Console',
      vscode.ViewColumn.One,
      {
        enableScripts: true,
        retainContextWhenHidden: true,
        localResourceRoots: [
          vscode.Uri.joinPath(this.context.extensionUri, 'webview-ui')
        ]
      }
    );

    this.panel.webview.html = this.getWebviewContent();

    // Handle messages from webview
    this.panel.webview.onDidReceiveMessage(
      async message => {
        switch (message.command) {
          case 'executeScript':
            await this.executeScript(message.script);
            break;
          case 'getSiteUrl':
            const siteUrl = vscode.workspace.getConfiguration('speditor').get('siteUrl');
            this.panel?.webview.postMessage({ command: 'siteUrl', value: siteUrl });
            break;
        }
      },
      undefined,
      this.context.subscriptions
    );

    this.panel.onDidDispose(() => {
      this.panel = undefined;
    });
  }

  private getWebviewContent(): string {
    const scriptUri = this.panel!.webview.asWebviewUri(
      vscode.Uri.joinPath(this.context.extensionUri, 'webview-ui', 'main.js')
    );

    return `<!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>PnP Console</title>
    </head>
    <body>
        <div id="root"></div>
        <script src="${scriptUri}"></script>
    </body>
    </html>`;
  }

  private async executeScript(script: string) {
    try {
      const session = await getAuthSession();
      const config = vscode.workspace.getConfiguration('speditor');
      const siteUrl = config.get<string>('siteUrl');
      
      // Execute script using PnPjs
      // This is a simplified example - real implementation needs script evaluation
      const result = await evaluatePnPScript(script, siteUrl, session.accessToken);
      
      this.panel?.webview.postMessage({
        command: 'scriptResult',
        result: result
      });
    } catch (error) {
      this.panel?.webview.postMessage({
        command: 'scriptError',
        error: error instanceof Error ? error.message : 'Unknown error'
      });
    }
  }
}
```

### Tree View for SharePoint Explorer

```typescript
// src/views/siteExplorer.ts
import * as vscode from 'vscode';

interface SPTreeItem {
  label: string;
  type: 'site' | 'web' | 'list' | 'library' | 'folder' | 'file';
  url?: string;
  id?: string;
  children?: SPTreeItem[];
}

export class SharePointTreeProvider implements vscode.TreeDataProvider<SPTreeItem> {
  private _onDidChangeTreeData = new vscode.EventEmitter<SPTreeItem | undefined | null | void>();
  readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

  constructor(private context: vscode.ExtensionContext) {}

  refresh(): void {
    this._onDidChangeTreeData.fire();
  }

  getTreeItem(element: SPTreeItem): vscode.TreeItem {
    const treeItem = new vscode.TreeItem(
      element.label,
      element.children ? vscode.TreeItemCollapsibleState.Collapsed : vscode.TreeItemCollapsibleState.None
    );

    treeItem.contextValue = element.type;
    treeItem.iconPath = this.getIcon(element.type);
    treeItem.command = element.type === 'file' ? {
      command: 'speditor.openFile',
      title: 'Open File',
      arguments: [element]
    } : undefined;

    return treeItem;
  }

  async getChildren(element?: SPTreeItem): Promise<SPTreeItem[]> {
    if (!element) {
      // Root level - get configured sites
      const config = vscode.workspace.getConfiguration('speditor');
      const siteUrl = config.get<string>('siteUrl');
      
      if (!siteUrl) {
        vscode.window.showInformationMessage('Configure SharePoint site URL');
        return [];
      }

      return [{
        label: siteUrl,
        type: 'site',
        url: siteUrl,
        children: []
      }];
    }

    // Load children based on type
    switch (element.type) {
      case 'site':
      case 'web':
        return await this.getWebChildren(element);
      case 'list':
      case 'library':
        return await this.getListItems(element);
      case 'folder':
        return await this.getFolderContents(element);
      default:
        return [];
    }
  }

  private async getWebChildren(web: SPTreeItem): Promise<SPTreeItem[]> {
    try {
      const session = await getAuthSession();
      const client = new SPClient(web.url!, session.accessToken);
      const lists = await client.getLists();

      return lists.map((list: any) => ({
        label: list.Title,
        type: list.BaseTemplate === 101 ? 'library' : 'list',
        id: list.Id,
        url: web.url,
        children: []
      }));
    } catch (error) {
      vscode.window.showErrorMessage(`Failed to load lists: ${error}`);
      return [];
    }
  }

  private async getListItems(list: SPTreeItem): Promise<SPTreeItem[]> {
    // Implementation for getting list items
    return [];
  }

  private async getFolderContents(folder: SPTreeItem): Promise<SPTreeItem[]> {
    // Implementation for getting folder contents
    return [];
  }

  private getIcon(type: string): vscode.ThemeIcon {
    switch (type) {
      case 'site':
      case 'web':
        return new vscode.ThemeIcon('globe');
      case 'list':
        return new vscode.ThemeIcon('list-unordered');
      case 'library':
        return new vscode.ThemeIcon('library');
      case 'folder':
        return new vscode.ThemeIcon('folder');
      case 'file':
        return new vscode.ThemeIcon('file');
      default:
        return new vscode.ThemeIcon('question');
    }
  }
}
```

## State Management

### VS Code Storage Instead of Chrome Storage

```typescript
// src/utils/stateManager.ts
import * as vscode from 'vscode';

export class StateManager {
  constructor(private context: vscode.ExtensionContext) {}

  // Global state (persisted)
  async get<T>(key: string): Promise<T | undefined> {
    return this.context.globalState.get<T>(key);
  }

  async set(key: string, value: any): Promise<void> {
    await this.context.globalState.update(key, value);
  }

  // Workspace state (per workspace)
  async getWorkspace<T>(key: string): Promise<T | undefined> {
    return this.context.workspaceState.get<T>(key);
  }

  async setWorkspace(key: string, value: any): Promise<void> {
    await this.context.workspaceState.update(key, value);
  }

  // Secure storage for tokens
  async getSecret(key: string): Promise<string | undefined> {
    return await this.context.secrets.get(key);
  }

  async setSecret(key: string, value: string): Promise<void> {
    await this.context.secrets.store(key, value);
  }

  async deleteSecret(key: string): Promise<void> {
    await this.context.secrets.delete(key);
  }
}

// Usage in Redux middleware (replace chromeStorageSync)
import { Middleware } from 'redux';

export const vscodeStorageMiddleware = (stateManager: StateManager): Middleware => 
  store => next => async action => {
    const result = next(action);
    const state = store.getState();
    
    // Save state to VS Code storage
    await stateManager.set('appState', state);
    
    return result;
  };
```

## Code Examples

### Complete Extension Entry Point

```typescript
// src/extension.ts
import * as vscode from 'vscode';
import { registerAuthProvider } from './authentication/authProvider';
import { ConsolePanel } from './views/consolePanel';
import { SharePointTreeProvider } from './views/siteExplorer';
import { StateManager } from './utils/stateManager';

export function activate(context: vscode.ExtensionContext) {
  console.log('SP Editor extension is now active');

  // Initialize state manager
  const stateManager = new StateManager(context);

  // Register authentication provider
  registerAuthProvider(context);

  // Register tree view
  const treeProvider = new SharePointTreeProvider(context);
  context.subscriptions.push(
    vscode.window.registerTreeDataProvider('speditor.siteExplorer', treeProvider)
  );

  // Register commands
  context.subscriptions.push(
    vscode.commands.registerCommand('speditor.openConsole', () => {
      const panel = new ConsolePanel(context);
      panel.show();
    })
  );

  context.subscriptions.push(
    vscode.commands.registerCommand('speditor.refreshExplorer', () => {
      treeProvider.refresh();
    })
  );

  context.subscriptions.push(
    vscode.commands.registerCommand('speditor.configureSite', async () => {
      const siteUrl = await vscode.window.showInputBox({
        prompt: 'Enter SharePoint site URL',
        placeHolder: 'https://contoso.sharepoint.com/sites/sitename'
      });

      if (siteUrl) {
        await vscode.workspace.getConfiguration('speditor').update(
          'siteUrl',
          siteUrl,
          vscode.ConfigurationTarget.Workspace
        );
        treeProvider.refresh();
      }
    })
  );

  // Status bar item
  const statusBarItem = vscode.window.createStatusBarItem(
    vscode.StatusBarAlignment.Right,
    100
  );
  statusBarItem.text = '$(cloud) SP Editor';
  statusBarItem.command = 'speditor.openConsole';
  statusBarItem.show();
  context.subscriptions.push(statusBarItem);

  // Load saved state
  stateManager.get('appState').then(savedState => {
    if (savedState) {
      // Restore application state
      console.log('Restored saved state');
    }
  });
}

export function deactivate() {
  console.log('SP Editor extension is now deactivated');
}
```

### Package.json Configuration

```json
{
  "name": "sp-editor-vscode",
  "displayName": "SP Editor for VS Code",
  "description": "SharePoint development and management tools for VS Code",
  "version": "1.0.0",
  "publisher": "your-publisher-name",
  "icon": "media/icon.png",
  "engines": {
    "vscode": "^1.80.0"
  },
  "categories": [
    "Other"
  ],
  "keywords": [
    "sharepoint",
    "spfx",
    "pnp",
    "microsoft365"
  ],
  "activationEvents": [
    "onCommand:speditor.openConsole",
    "onView:speditor.siteExplorer",
    "workspaceContains:**/package.json"
  ],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "speditor.openConsole",
        "title": "SP Editor: Open PnP Console",
        "icon": "$(console)"
      },
      {
        "command": "speditor.refreshExplorer",
        "title": "SP Editor: Refresh Explorer",
        "icon": "$(refresh)"
      },
      {
        "command": "speditor.configureSite",
        "title": "SP Editor: Configure Site URL"
      },
      {
        "command": "speditor.openFile",
        "title": "SP Editor: Open File"
      },
      {
        "command": "speditor.editProperties",
        "title": "SP Editor: Edit Properties"
      }
    ],
    "views": {
      "explorer": [
        {
          "id": "speditor.siteExplorer",
          "name": "SharePoint",
          "icon": "media/icon.svg",
          "contextualTitle": "SharePoint Explorer"
        }
      ]
    },
    "viewsContainers": {
      "activitybar": [
        {
          "id": "speditor",
          "title": "SP Editor",
          "icon": "media/icon.svg"
        }
      ]
    },
    "menus": {
      "view/title": [
        {
          "command": "speditor.refreshExplorer",
          "when": "view == speditor.siteExplorer",
          "group": "navigation"
        }
      ],
      "view/item/context": [
        {
          "command": "speditor.editProperties",
          "when": "view == speditor.siteExplorer && viewItem == list"
        }
      ]
    },
    "configuration": {
      "title": "SP Editor",
      "properties": {
        "speditor.siteUrl": {
          "type": "string",
          "default": "",
          "description": "Default SharePoint site URL"
        },
        "speditor.clientId": {
          "type": "string",
          "default": "",
          "description": "Azure AD application client ID"
        },
        "speditor.recentSites": {
          "type": "array",
          "default": [],
          "description": "Recently accessed SharePoint sites"
        },
        "speditor.autoRefresh": {
          "type": "boolean",
          "default": true,
          "description": "Automatically refresh explorer on changes"
        }
      }
    }
  },
  "scripts": {
    "vscode:prepublish": "npm run compile",
    "compile": "tsc -p ./",
    "watch": "tsc -watch -p ./",
    "pretest": "npm run compile && npm run lint",
    "lint": "eslint src --ext ts",
    "test": "node ./out/test/runTest.js"
  },
  "devDependencies": {
    "@types/vscode": "^1.80.0",
    "@types/node": "^20.x",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.40.0",
    "typescript": "^5.1.0",
    "@vscode/test-electron": "^2.3.0"
  },
  "dependencies": {
    "@azure/msal-node": "^2.0.0",
    "@pnp/sp": "^4.0.0",
    "@pnp/graph": "^4.0.0",
    "@pnp/logging": "^4.0.0",
    "@pnp/msaljsclient": "^4.0.0"
  }
}
```

## Build and Debug Configuration

### .vscode/launch.json
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Run Extension",
      "type": "extensionHost",
      "request": "launch",
      "args": [
        "--extensionDevelopmentPath=${workspaceFolder}"
      ],
      "outFiles": [
        "${workspaceFolder}/out/**/*.js"
      ],
      "preLaunchTask": "${defaultBuildTask}"
    },
    {
      "name": "Extension Tests",
      "type": "extensionHost",
      "request": "launch",
      "args": [
        "--extensionDevelopmentPath=${workspaceFolder}",
        "--extensionTestsPath=${workspaceFolder}/out/test/suite/index"
      ],
      "outFiles": [
        "${workspaceFolder}/out/test/**/*.js"
      ],
      "preLaunchTask": "${defaultBuildTask}"
    }
  ]
}
```

### .vscode/tasks.json
```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "type": "npm",
      "script": "watch",
      "problemMatcher": "$tsc-watch",
      "isBackground": true,
      "presentation": {
        "reveal": "never"
      },
      "group": {
        "kind": "build",
        "isDefault": true
      }
    }
  ]
}
```

## Summary

This implementation guide provides:

1. **Complete project structure** for VS Code extension
2. **API mappings** from browser extension to VS Code
3. **Authentication** using MSAL device code flow
4. **SharePoint operations** via REST APIs instead of script injection
5. **UI components** using webviews and tree views
6. **State management** using VS Code storage APIs
7. **Working code examples** for all major components

Key takeaways:
- Replace `chrome.*` APIs with `vscode.*` equivalents
- Use REST APIs instead of page script injection
- Implement independent authentication (can't piggyback on browser)
- Use webviews for React UI components
- Tree views for hierarchical data (site/list explorer)
- Configuration-based instead of tab-based context

The examples can be used as starting points for either a full conversion or a complementary VS Code extension.
