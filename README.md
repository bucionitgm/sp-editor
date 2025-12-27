
# SP Editor for Microsoft Edge
This is total re-write of SP Editor Extension using React, Office UI Fabric and Ionic. This Extension is installable to Microsoft Edge from [Microsoft Edge Addons](https://microsoftedge.microsoft.com/addons/detail/affnnhcbfmcbbdlcadgkdbfafigmjdkk). Not all features has been ported yet, but I think feature parity will be set during H1. When that happens, Chrome SP Editor will be updated to the same code base.

![SP Editor](repo-images/edgespeditor.png)

If you want to chip in by porting features or even creating new ones, here is a quide how to get started contributing.

### running locally with watch mode
```powershell
git clone https://github.com/pnp/sp-editor.git # clone the project
cd sp-editor # go to the folder
code . # open vscode
npm i # install dependencies
npm run build # to build everything before starting to developing
npm start # build and start watch mode
```
When Watch is running, open Microsoft Edge and select Extensions from the menu

![](repo-images/edgemenu.png)

Enable Developer Mode

![](repo-images/edgedevelopermode.png)

Load Unpacked Extension, select the **build** folder of the project

![](repo-images/edgeloadunpacked.png)

If all good, the local build extension will show up

![](repo-images/edgeextensionloaded.png)

Now you can open a SharePoint site, open devtools and select SharePoint tab. The extension updates it self on file changes. If it does not, press the reload button to reload extension after making code changes.

![](repo-images/edgewatchrefresh.png)

To inspect the Extension, you can open the extension devtool by right clicking and selecting **Inspect** and you can see the dom/console/sources/etc of the extension.

![](repo-images/edgeinspect.png)

## VS Code Extension Analysis

If you're interested in converting this browser extension to a VS Code extension, please refer to the following documentation:

- **[VS Code Extension Feasibility Analysis](VSCODE_EXTENSION_ANALYSIS.md)** - Comprehensive analysis of converting SP Editor to VS Code extension, including challenges, architecture differences, and recommended approaches.

- **[Implementation Guide](VSCODE_IMPLEMENTATION_GUIDE.md)** - Technical implementation details, code examples, and API mappings for building a VS Code extension.

- **[Browser vs VS Code Comparison](BROWSER_VS_VSCODE_COMPARISON.md)** - Detailed feature comparison, use case scenarios, and recommendations on the best approach.

**TL;DR**: The analysis recommends creating a **complementary VS Code extension** that focuses on SharePoint development workflows while maintaining the browser extension for live page interaction and troubleshooting. Both tools serve different purposes and provide the best user experience when used together.
