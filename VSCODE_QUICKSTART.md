# Quick Start: Building a Complementary VS Code Extension

This guide helps you get started building a VS Code extension that complements the SP Editor browser extension.

## Prerequisites

- Node.js 20+ and npm 10+
- Visual Studio Code
- Basic TypeScript knowledge
- Familiarity with SharePoint and PnPjs

## Option 1: Start from Scratch (Recommended for Learning)

### Step 1: Create Extension Skeleton

```bash
# Install Yeoman and VS Code Extension generator
npm install -g yo generator-code

# Generate extension
yo code

# Choose:
# ? What type of extension do you want to create? New Extension (TypeScript)
# ? What's the name of your extension? sp-editor-vscode
# ? What's the identifier of your extension? sp-editor-vscode
# ? What's the description of your extension? SharePoint development tools
# ? Initialize a git repository? Yes
# ? Bundle the source code with webpack? No
# ? Which package manager to use? npm

cd sp-editor-vscode
```

### Step 2: Install Core Dependencies

```bash
npm install @pnp/sp @pnp/graph @pnp/logging @pnp/msaljsclient @azure/msal-node
npm install --save-dev @types/vscode
```

### Step 3: Create Basic Structure

```bash
mkdir -p src/authentication
mkdir -p src/sharepoint
mkdir -p src/views
mkdir -p src/commands
mkdir -p src/utils
```

### Step 4: Implement Hello World Feature

Edit `src/extension.ts`:

```typescript
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {
    console.log('SP Editor VS Code extension is active');

    // Register a simple command
    let disposable = vscode.commands.registerCommand(
        'sp-editor-vscode.helloSharePoint',
        async () => {
            const siteUrl = await vscode.window.showInputBox({
                prompt: 'Enter SharePoint site URL',
                placeHolder: 'https://contoso.sharepoint.com/sites/yoursite'
            });

            if (siteUrl) {
                vscode.window.showInformationMessage(
                    `SP Editor: Connected to ${siteUrl}`
                );
                
                // Store for later use
                await context.globalState.update('siteUrl', siteUrl);
            }
        }
    );

    context.subscriptions.push(disposable);

    // Add status bar item
    const statusBarItem = vscode.window.createStatusBarItem(
        vscode.StatusBarAlignment.Right,
        100
    );
    statusBarItem.text = '$(cloud) SP Editor';
    statusBarItem.command = 'sp-editor-vscode.helloSharePoint';
    statusBarItem.show();
    context.subscriptions.push(statusBarItem);
}

export function deactivate() {}
```

Update `package.json` contributes section:

```json
{
  "contributes": {
    "commands": [
      {
        "command": "sp-editor-vscode.helloSharePoint",
        "title": "SP Editor: Connect to SharePoint"
      }
    ]
  }
}
```

### Step 5: Test Your Extension

1. Press `F5` to launch Extension Development Host
2. In the new window, press `Ctrl+Shift+P`
3. Type "SP Editor: Connect to SharePoint"
4. Enter a SharePoint URL
5. You should see the status bar item and confirmation message

### Step 6: Add First Real Feature - Site Info

Create `src/sharepoint/client.ts`:

```typescript
import { spfi, SPFx } from '@pnp/sp';
import '@pnp/sp/webs';

export interface ISiteInfo {
    title: string;
    url: string;
    description: string;
    created: Date;
}

export class SharePointClient {
    private sp: any;

    constructor(siteUrl: string, accessToken?: string) {
        // For now, we'll use anonymous access (limited)
        // Later: add authentication
        this.sp = spfi(siteUrl);
    }

    async getSiteInfo(): Promise<ISiteInfo> {
        try {
            const web = await this.sp.web
                .select('Title', 'Url', 'Description', 'Created')();
            
            return {
                title: web.Title,
                url: web.Url,
                description: web.Description,
                created: new Date(web.Created)
            };
        } catch (error) {
            throw new Error(`Failed to get site info: ${error}`);
        }
    }
}
```

Add command in `src/extension.ts`:

```typescript
// Add this to activate function
let getSiteInfoCommand = vscode.commands.registerCommand(
    'sp-editor-vscode.getSiteInfo',
    async () => {
        const siteUrl = context.globalState.get<string>('siteUrl');
        
        if (!siteUrl) {
            vscode.window.showWarningMessage(
                'Please connect to SharePoint first'
            );
            return;
        }

        try {
            vscode.window.withProgress(
                {
                    location: vscode.ProgressLocation.Notification,
                    title: 'Loading site information...',
                    cancellable: false
                },
                async () => {
                    const client = new SharePointClient(siteUrl);
                    const info = await client.getSiteInfo();
                    
                    const message = `Site: ${info.title}\n` +
                                  `URL: ${info.url}\n` +
                                  `Created: ${info.created.toLocaleDateString()}`;
                    
                    vscode.window.showInformationMessage(message);
                }
            );
        } catch (error) {
            vscode.window.showErrorMessage(
                `Error: ${error instanceof Error ? error.message : 'Unknown error'}`
            );
        }
    }
);

context.subscriptions.push(getSiteInfoCommand);
```

## Option 2: Port Existing Features

If you want to port features from the browser extension to VS Code:

### Step 1: Choose a Feature to Port

Start with a simple feature like "List Properties" which doesn't require complex UI.

### Step 2: Extract Business Logic

Copy the SharePoint operation logic from browser extension:

From: `src/pages/listproperties/chrome/getlistproperties.ts`

To: `src/sharepoint/operations/listProperties.ts`

### Step 3: Replace Chrome APIs

**Before (Browser Extension):**
```typescript
chrome.scripting.executeScript({
    target: { tabId: chrome.devtools.inspectedWindow.tabId },
    world: 'MAIN',
    args: [listId],
    func: getListProperties
});
```

**After (VS Code Extension):**
```typescript
const client = new SharePointClient(siteUrl, accessToken);
const properties = await client.getListProperties(listId);
```

### Step 4: Adapt UI

**Before (Browser Extension):** React component in DevTools panel

**After (VS Code Extension):** Choose one:
1. Tree view for hierarchical data
2. Webview panel for complex UI
3. Quick pick for simple selection
4. Information message for results

### Step 5: Test and Iterate

## Option 3: Minimal Viable Extension (MVP)

Build a minimal extension with just the most valuable features:

### Recommended MVP Features:

1. **Site Configuration** ✅
   - Store SharePoint site URL
   - Quick site switching
   - Recent sites list

2. **PnP Console** ✅
   - Execute PnPjs code
   - Show results
   - Save scripts

3. **File Explorer** ✅
   - Browse document libraries
   - Download files to workspace
   - Upload files from workspace

4. **Property Editor** ✅
   - View/edit site properties
   - View/edit list properties

### Time Estimate for MVP:
- 2 weeks for experienced VS Code extension developer
- 4 weeks for someone new to VS Code extensions

## Development Workflow

### Daily Development:

```bash
# Terminal 1: Watch for TypeScript changes
npm run watch

# Terminal 2: Test extension (or use F5 in VS Code)
# Press F5 to launch Extension Development Host

# Make changes, save, reload extension with Ctrl+R in Extension Host
```

### Debugging:

1. Set breakpoints in your TypeScript code
2. Press F5 to launch Extension Development Host
3. Trigger your command
4. Debugger will pause at breakpoints

### Testing:

```bash
# Run tests
npm test

# Add test file: src/test/suite/extension.test.ts
```

## Common Patterns

### Pattern 1: Command with Configuration

```typescript
vscode.commands.registerCommand('extension.command', async () => {
    const config = vscode.workspace.getConfiguration('speditor');
    const siteUrl = config.get<string>('siteUrl');
    
    if (!siteUrl) {
        const input = await vscode.window.showInputBox({
            prompt: 'Enter site URL'
        });
        
        if (input) {
            await config.update('siteUrl', input, vscode.ConfigurationTarget.Global);
        }
    }
    
    // Use siteUrl...
});
```

### Pattern 2: Tree View Provider

```typescript
class MyTreeProvider implements vscode.TreeDataProvider<TreeItem> {
    private _onDidChangeTreeData = new vscode.EventEmitter<TreeItem | undefined>();
    readonly onDidChangeTreeData = this._onDidChangeTreeData.event;

    refresh(): void {
        this._onDidChangeTreeData.fire();
    }

    getTreeItem(element: TreeItem): vscode.TreeItem {
        return element;
    }

    async getChildren(element?: TreeItem): Promise<TreeItem[]> {
        // Return child items
        return [];
    }
}

// Register
const treeProvider = new MyTreeProvider();
vscode.window.registerTreeDataProvider('myView', treeProvider);
```

### Pattern 3: Webview Panel

```typescript
function createWebviewPanel(context: vscode.ExtensionContext) {
    const panel = vscode.window.createWebviewPanel(
        'myPanel',
        'My Panel',
        vscode.ViewColumn.One,
        { enableScripts: true }
    );

    panel.webview.html = getWebviewContent();

    // Receive messages from webview
    panel.webview.onDidReceiveMessage(
        message => {
            switch (message.command) {
                case 'alert':
                    vscode.window.showInformationMessage(message.text);
                    return;
            }
        },
        undefined,
        context.subscriptions
    );

    // Send message to webview
    panel.webview.postMessage({ command: 'update', data: {} });
}
```

### Pattern 4: Progress Notification

```typescript
vscode.window.withProgress(
    {
        location: vscode.ProgressLocation.Notification,
        title: 'Loading...',
        cancellable: true
    },
    async (progress, token) => {
        token.onCancellationRequested(() => {
            console.log('User cancelled');
        });

        progress.report({ increment: 0 });

        // Do work...
        await doWork();

        progress.report({ increment: 100 });
    }
);
```

## Troubleshooting

### Issue: "Cannot find module '@pnp/sp'"

**Solution:**
```bash
npm install @pnp/sp @pnp/graph @pnp/logging
```

### Issue: "Command not found"

**Solution:** Check `package.json` contributes section has the command registered.

### Issue: "Extension not activating"

**Solution:** Check `activationEvents` in `package.json`:
```json
{
  "activationEvents": [
    "onCommand:extension.command",
    "onView:myView"
  ]
}
```

### Issue: SharePoint API returns 401 Unauthorized

**Solution:** You need to implement authentication. See [VSCODE_IMPLEMENTATION_GUIDE.md](VSCODE_IMPLEMENTATION_GUIDE.md) for authentication setup.

### Issue: Can't access SharePoint context

**Solution:** VS Code extensions can't access browser tabs or page context. You need to:
1. Get site URL from configuration
2. Make REST API calls
3. Cannot access `window._spPageContextInfo`

## Next Steps

1. ✅ **Complete the MVP** - Get basic features working
2. ✅ **Add Authentication** - Implement MSAL authentication
3. ✅ **Build More Features** - Port features from browser extension
4. ✅ **Add Tests** - Write unit and integration tests
5. ✅ **Polish UI** - Improve user experience
6. ✅ **Documentation** - Write user guide
7. ✅ **Publish** - Publish to VS Code Marketplace

## Resources

- [VS Code Extension API](https://code.visualstudio.com/api)
- [PnPjs Documentation](https://pnp.github.io/pnpjs/)
- [MSAL Node](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-node)
- [Extension Samples](https://github.com/microsoft/vscode-extension-samples)
- [Publishing Extensions](https://code.visualstudio.com/api/working-with-extensions/publishing-extension)

## Getting Help

If you get stuck:

1. Check [VSCODE_IMPLEMENTATION_GUIDE.md](VSCODE_IMPLEMENTATION_GUIDE.md) for detailed examples
2. Review [VS Code Extension Samples](https://github.com/microsoft/vscode-extension-samples)
3. Ask in [VS Code Extension Development Discussion](https://github.com/microsoft/vscode/discussions)
4. Check [PnP Community](https://pnp.github.io/)

## Contributing Back

If you build features that would benefit others:

1. Fork the repository
2. Create a feature branch
3. Implement and test your feature
4. Submit a pull request
5. Add documentation

Good luck building your VS Code extension! 🚀
