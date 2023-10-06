vscode的代码补全等高级功能是通过language Api来进行实现的。<br />vscode的language api有三要素：<br />事件，参数，响应体。
<a name="GCHm6"></a>
# 如何创建带languageApi的插件？
1：输入yo code并选择new ts项目<br />2：加入一个入口文件和activationEvents的配置项<br />activationEvent：作用是告诉vscode什么时候激活该插件<br />main：作用是指明入口文件<br />3：在contribute项目下创建一个language配置项和gramars配置项。
```json
"languages": [
  {
    "id": "ux",
    "aliases": [
      "ux",
      "ux"
    ],
    "extensions": [
      ".ux"
    ],
    "configuration": "./language-configuration.json"
  }
],
"grammars": [
  {
    "language": "ux",
    "scopeName": "source.ux",
    "path": "./syntaxes/ux.tmLanguage.json"
  }
]
```
具体文件并不重要

<a name="YlWOe"></a>
# LSP（language server protocol）
vscode会代理一些基本的用户行为（鼠标移动，点击，双击）等。当触发该行为时，会经过事件返回到代码中解析为事件 。<br />如果需要使用lsp的话，建议先安装官方示例。
```json
https://code.visualstudio.com/api/language-extensions/language-server-extension-guide
```
 步骤1：<br />关键2个文件夹	
```json
.
├── client // Language Client
│   ├── src
│   │   └── extension.ts // Language Client 入口文件
├── package.json
└── server // Language Server
    └── src
        └── server.ts // Language Server 入口文件
```
client文件夹表示入口文件<br />server文件夹表示主要服务文件，关键的代码写在此处。<br />步骤2：<br />在server.ts文件中启动一个服务的关键步骤<br />1：从2个关键包中引入对象。初始化一个lsp连接对象
```javascript
import {
	createConnection,
	TextDocuments,
	Diagnostic,
	DiagnosticSeverity,
	ProposedFeatures,
	InitializeParams,
	DidChangeConfigurationNotification,
	CompletionItem,
	CompletionItemKind,
	TextDocumentPositionParams,
	TextDocumentSyncKind,
	InitializeResult
} from 'vscode-languageserver/node';

import {
	TextDocument
} from 'vscode-languageserver-textdocument';
const connection = createConnection(ProposedFeatures.all);
```
<br />2：将文件对象映射到本地
```javascript
const documents: TextDocuments<TextDocument> = new TextDocuments(TextDocument);
```
3：明确声明插件支持的语言特性列表
```javascript
connection.onInitialize((params: InitializeParams) => {
  // 明确声明插件所支持的语言特性
	const result: InitializeResult = {
		capabilities: {
			textDocumentSync: TextDocumentSyncKind.Incremental,
			// Tell the client that this server supports code completion.
			completionProvider: {
				resolveProvider: true
			}
		}
	};
	if (hasWorkspaceFolderCapability) {
		result.capabilities.workspace = {
			workspaceFolders: {
				supported: true
			}
		};
	}
	return result;
});
```
<a name="pGYpf"></a>
# vscode插件开发：
<br />原因：原版ide因为自带语法校验插件，且支持度不够。且并未更新。需要更改其中的校验规则。目前使用插件来进行定位校验规则并进行修改。<br />原理：在触发了诊断的监听中。对取得的诊断进行分析。检索到规定字段后执行函数。获取到目标语法校验插件的具体路径。对语法插件的校验文件内容进行具体修改。
<a name="X8ZPI"></a>
## vscode的关键理解
一：vscode的插件如果不适用lsp来进行开发。那就是由package.json声明的入口文件进入。并按照从上至下的顺序执行入口文件的代码。<br />二：vscode的实例都可以由`vscode`这个依赖来进行获取。内部可以根据其得到具体的vscode实例以及方法。<br />三：vscode的入口文件使用activate方法来进行创建插件的进程。并在activate方法内获取到具体的context对象。这个对象目前就是插件实例。指向的是当前的插件<br />四：插件实例和vscode实例指向的是不同的实例。`context.subscription.push`这个方法指的是向当前的插件进程推送监听事件，比如`context.subscription.push(vscode.languages.onDidChangeDiagnostics())`这就是让当前的插件监听代码诊断的触发<br />五：需要注意的是由于闭包的存在。每一次监听事件的创建实际上都形成了闭包。所以尽量避免不同的代码操纵同一个对象

<a name="BlBgB"></a>
## vscode检查目标插件地址
使用`vscode.extension.getExtenxion()`方法可以获取到目标插件的具体实例。该实例有一个方法是获得插件的地址。`extensionPath()`<br />第二步是根据插件地址拼接到具体的文件地址。<br />第三步是使用nodejs自带的fs方法去读取文件。在此需要使用`fs.readFile()`和`fs.writeFile()`<br />注意：由于`fs.readFile()`和`fs.writeFile()`是同步行为。会对同时对文件操作。所以需要把操纵行为封装为promise函数并在外部使用await来进行覆盖。

<a name="r171o"></a>
# vscode lsp插件开发流程如下
1：使用yo code生成插件的基础架构。如果需要使用的是针对不同语言的调试。需要在package.json中引入对不同语言的支持

2：创建`client/src/extension.ts`来创建链接<br />3：创建`client/src/server.ts`来创建服务器<br />4：在服务端的文件下创建capabilities中声明特性支持
```javascript
// 关键点1： 初始化 LSP 连接对象
const connection = createConnection(ProposedFeatures.all);

// 关键点2： 创建文档集合对象，用于映射到实际文档
const documents: TextDocuments<TextDocument> = new TextDocuments(TextDocument);
connection.onInitialize((params: InitializeParams) => {
  // 明确声明插件支持的语言特性
  const result: InitializeResult = {
    capabilities: {
      // 增量处理
      textDocumentSync: TextDocumentSyncKind.Incremental,
      // 代码补全
      completionProvider: {
        resolveProvider: true,
      },
      // hover 提示
      hoverProvider: true,
      // 签名提示
      signatureHelpProvider: {
        triggerCharacters: ["("],
      },
      // 格式化
      documentFormattingProvider: true,
      // 语言高亮
      documentHighlightProvider: true,
    },
  };
  return result;
});
```
5：在服务端中注册事件，并在事件中拿到对代码的语义化截取。<br />6：启动调试。注意需要选择的是client+server调试。否则无法使用debugger<br />注意点：需要注意的几点：<br />1：运行插件会无法找到client+server调试。需要配置vscode的launch.json文件
```json
// A launch configuration that compiles the extension and then opens it inside a new window
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "extensionHost",
      "request": "launch",
      "name": "Launch Client",
      "runtimeExecutable": "${execPath}",
      "args": [
        "--extensionDevelopmentPath=${workspaceRoot}",
        "--disable-extensions",
        "${workspaceRoot}/test_files"
      ],
      "outFiles": ["${workspaceRoot}/client/out/**/*.js"],
      "smartStep": true,
      "preLaunchTask": {
        "type": "npm",
        "script": "watch"
      }
    },
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to Server",
      "port": 6009,
      "restart": true,
      "outFiles": ["${workspaceRoot}/server/out/**/*.js"],
      "preLaunchTask": {
        "type": "npm",
        "script": "watch"
      }
    }
  ],
  "compounds": [
    {
      "name": "Client + Server",
      "configurations": ["Launch Client", "Attach to Server"]
    }
  ]
}

```
2：vscode无法解析client和server目录下的ts文件。也无法编译。这也是vscode的配置问题原因。需要打开task.json并配置
```json
		{
			"type": "typescript",
			"tsconfig": "tsconfig.json",
			"problemMatcher": [
				"$tsc"
			],
			"group": "build",
			"label": "tsc: 构建 - tsconfig.json"
		}
```
加入这一条就可以被任务识别。
<a name="rJzj0"></a>
# vscode的lsp和插件区别
vscode的lsp提供诊断，代码自动补全，格式化等操作。<br />例如，如果我需要实现一个抓取所有的诊断实例的功能。就需要在client下进行操作。因为client下才能获取到不是lsp的api和所有的诊断对象<br />例如，如果我要根据输入来监听到文本内容，并抛出诊断，那么则需要在sever下，即lsp下进行。lsp下的api基本上都是基于文本对象和当前的文档对象。而client下则有更加底层的api。

<a name="at5ua"></a>
# VSCODE如何根据错误来进行快速修复
理解几个点：1：诊断信息是与文档进行绑定的，在client下可以获取到当前文档的所有诊断信息。<br />2：诊断信息内包含了range和uri等信息，可以被client获取到。<br />3：快速修复需要提供一个command，并需要在package.json中进行注册<br />4：根据不同的command可以提供不同的快速修复选项<br />理解了以上点以后就可以进行<br />1：第一步，使用api进行注册command命令和指定的行为<br />`vscode.commands.registerCommand`api是进行注册的command的。参数会抛出引用他的对象<br />2：使用`vscode.languages.registerCodeActionsProvider`api来进行注册command对应的行为。<br />该api接受一个对应的文件列表，一个处理程序，一个包含有quickfix字段的元数据<br />3：文件列表和元数据都比较好处理，我们优先进行把处理程序封装的函数解决
```javascript
export function autoFix(context: vscode.ExtensionContext) {
  context.subscriptions.push(blankQuickFixAction())//注册命令，返回一个由vscode.commands.registerCommand创建的回调
  context.subscriptions.push(vscode.languages.registerCodeActionsProvider(
    file,
    new CodeActionProvider(),//该函数的主要目的就是为了注册quickfix项，并且给quickfix添加备注以及对应的command
    new CodeActionProviderMetadata()
  ));
}
```
该函数逻辑如下：1：进行筛选，筛选出需要的诊断对象<br />2：使用`new vscode.CodeAction`来创建出需要的对象，并将其返回出去，例如`new vscode.CodeAction('去除display项', vscode.CodeActionKind.QuickFix);`<br />3：给对象添加上title,command,arguments等属性，该处的arguments最终会被`vscode.commands.registerCommand`的参数给拿到。

<a name="rgb2j"></a>
# vscode的snippets
语法片段的开发相对简单，只需遵循以下步骤即可。<br />1：先进入package.json的contribute配置项下添加snippets字段<br />2：在根目录下创建snippets文件夹，该处即是代码片段储存位置<br />3：使用`[https://snippet-generator.app/?description=&tabtrigger=&snippet=&mode=vscode](https://snippet-generator.app/?description=&tabtrigger=&snippet=&mode=vscode)`生成对应的json文件来放如snippets文件夹内


<a name="rHr9c"></a>
# vscode的自动引入
如何使用vscode的自动引入？<br />vscode提供一个官方的api。可以根据自动输入的不同来决定右侧菜单栏的内容。右侧菜单的内容是自动联想。
```javascript
context.subscriptions.push(vscode.languages.registerCompletionItemProvider(file, new CompletionItemProvider(), '.'))
```
`registerCompletionItemProvider`这个监听器接受3个参数，支持的文件列表，回调，和启动字符串<br />文件列表和启动字符串不重要。重要的是回调。<br />回调必须由2个函数组成。而且这两个函数有固定的返回值：<br />provideCompletionItems：该函数需要返回一个由`CompletionItem`组成的数组。`CompletionItem`是一个字符联想的对象。通过输入特定的字符将匹配上的`CompletionItem`填入右侧菜单<br />resolveCompletionItem：选中一个`CompletionItem`产生的回调<br />但是该代码有一个重要的问题就是无法区分选中的具体是哪个CompletionItem对象，而且如果在右键菜单上下选择则会多次触发对象。
```javascript
class CompletionItemProvider {
    provideCompletionItems(document: vscode.TextDocument, position: vscode.Position, token: vscode.CancellationToken, context: vscode.CompletionContext) {
        // 支持换行 代码从起始位置到输入位置
        const text = document.getText(new vscode.Range(
            new vscode.Position(0, 0),
            position
        ));

        // 只有tyc_test调用会触发联想内容
        if(/tyc_test\.$/.test(text)){
            return doc.map(item => {
                return item.body.map(iitem => {
                    let completionItem = new vscode.CompletionItem(iitem, vscode.CompletionItemKind[item.kind]);
                    completionItem.detail=item.detail;
                    completionItem.documentation = item.documentation;
                    // 代码替换位置，查找位置会同步应用
                    completionItem.range = new vscode.Range(new vscode.Position(position.line, position.character), new vscode.Position(position.line, position.character));
                    return completionItem;
                });
            }).flat();
        }
    }

    // resolveCompletionItem(){}
}

export default function autoCompletion(context: vscode.ExtensionContext) {
    context.subscriptions.push(vscode.languages.registerCompletionItemProvider(file, new CompletionItemProvider(), '.'));
}
```
上部分代码是一个典型的右键菜单选项<br />可以通过给`CompletionItem`对象添加command方法来把对应的值传递给外部监听，并由监听command的方法来进行自动import<br />给CompletionItem对象添加command属性，知道右键菜单具体选中的值
```javascript
let path = (imp: ImportObject) => {
            if ((<any>imp.file).discovered) {
                return imp.file.fsPath;
            } else {
                let rp = PathHelper.normalisePath(
                    PathHelper.getRelativePath(context.document.uri.fsPath, imp.file.fsPath));
                return rp;
            }
        };

        let handlers = [];
        context.imports.forEach(i => {
            handlers.push({
                title: `[AI] Import ${i.name} from ${path(i)}`,
                command: 'extension.fixImport',
                arguments: [context.document, context.range, context.context, context.token, context.imports]
            });
        });
```
注册command方法：
```javascript
let fixer = vscode.commands.registerCommand('extension.resolveImport', (args) => {
            new ImportFixer().fix(args.document, undefined, undefined, undefined, [args.imp]);
        });
        context.subscriptions.push(fixer);
```
来到了command方法后，我们可以使用方法来进行操作文本结构了，因为改事件只由我们选中了右侧菜单中的数据才可以触发
```javascript
vscode.window.activeTextEditor.edit((editBuilder)=>{
				//插入指定位置
				new vscode.Position()
				editBuilder.insert()
			})
```
该方法有类似于insert，replace等方法，都是接受一个position参数和一个文本，进行文本内的替换。


<a name="zbaAw"></a>
# Vscode获取当前工作区
```javascript
/**
 * 获取当前所在工程根目录，有3种使用方法：<br>
 * getProjectPath(uri) uri 表示工程内某个文件的路径<br>
 * getProjectPath(document) document 表示当前被打开的文件document对象<br>
 * getProjectPath() 会自动从 activeTextEditor 拿document对象，如果没有拿到则报错
 * @param {*} document 
 */
getProjectPath(document) {
    if (!document) {
        document = vscode.window.activeTextEditor ? vscode.window.activeTextEditor.document : null;
    }
    if (!document) {
        this.showError('当前激活的编辑器不是文件或者没有文件被打开！');
        return '';
    }
    const currentFile = (document.uri ? document.uri : document).fsPath;
    let projectPath = null;

    let workspaceFolders = vscode.workspace.workspaceFolders.map(item => item.uri.path);
    // 由于存在Multi-root工作区，暂时没有特别好的判断方法，先这样粗暴判断
    // 如果发现只有一个根文件夹，读取其子文件夹作为 workspaceFolders
    if (workspaceFolders.length == 1 && workspaceFolders[0] === vscode.workspace.rootPath) {
        const rootPath = workspaceFolders[0];
        var files = fs.readdirSync(rootPath);
        workspaceFolders = files.filter(name => !/^\./g.test(name)).map(name => path.resolve(rootPath, name));
        // vscode.workspace.rootPath会不准确，且已过时
        // return vscode.workspace.rootPath + '/' + this._getProjectName(vscode, document);
    }
    workspaceFolders.forEach(folder => {
        if (currentFile.indexOf(folder) === 0) {
            projectPath = folder;
        }
    })
    if (!projectPath) {
        this.showError('获取工程根路径异常！');
        return '';
    }
    return projectPath;
},

```
