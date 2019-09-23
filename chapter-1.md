# VSCode源码分析 - 主启动流程
## 目录
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">简介</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">技术架构</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">启动主进程</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-2.md">实例化服务</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-3.md">事件分发</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-4.md">进程通信</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-5.md">主要窗口</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-6.md">开发调试</a>

## <a name="1">简介</a>
Visual Studio Code(简称VSCode) 是开源免费的IDE编辑器，原本是微软内部使用的云编辑器(Monaco)。

git仓库地址： https://github.com/microsoft/vscode

通过Eletron集成了桌面应用，可以跨平台使用，开发语言主要采用微软自家的TypeScript。
整个项目结构比较清晰，方便阅读代码理解。成为了最流行跨平台的桌面IDE应用

微软希望VSCode在保持核心轻量级的基础上，增加项目支持，智能感知，编译调试。
![img](https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/vscode-vside.png)

## 编译安装
下载最新版本,目前我用的是1.37.1版本
官方的wiki中有编译安装的说明  [How to Contribute](https://github.com/microsoft/vscode/wiki/How-to-Contribute?_blank)

Linux, Window, MacOS三个系统编译时有些差别，参考官方文档，
在编译安装依赖时如果遇到connect timeout, 需要进行科学上网。

> 需要注意的一点 运行环境依赖版本 Nodejs x64 version >= 10.16.0, < 11.0.0,  python 2.7(3.0不能正常执行)


## <a name="2">技术架构</a>
![img](https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/vscode-source.png?raw=true)

### Electron 

> [Electron](https://electronjs.org/?_blank) 是一个使用 JavaScript, HTML 和 CSS 等 Web 技术创建原生程序的框架，它负责比较难搞的部分，你只需把精力放在你的应用的核心上即可 (Electron = Node.js + Chromium + Native API)

### Monaco Editor
> [Monaco Editor](https://github.com/microsoft/monaco-editor?_blank)是微软开源项目, 为VS Code提供支持的代码编辑器，运行在浏览器环境中。编辑器提供代码提示，智能建议等功能。供开发人员远程更方便的编写代码，可独立运行。

### TypeScript
> TypeScript是一种由微软开发的自由和开源的编程语言。它是JavaScript的一个超集，而且本质上向这个语言添加了可选的静态类型和基于类的面向对象编程

## 目录结构
```
├── build         # gulp编译构建脚本
├── extensions    # 内置插件
├── product.json  # App meta信息
├── resources     # 平台相关静态资源
├── scripts       # 工具脚本，开发/测试
├── src           # 源码目录
└── typings       # 函数语法补全定义
└── vs
    ├── base        # 通用工具/协议和UI库
    │   ├── browser # 基础UI组件，DOM操作
    │   ├── common  # diff描述，markdown解析器，worker协议，各种工具函数
    │   ├── node    # Node工具函数
    │   ├── parts   # IPC协议（Electron、Node），quickopen、tree组件
    │   ├── test    # base单测用例
    │   └── worker  # Worker factory和main Worker（运行IDE Core：Monaco）
    ├── code        # VSCode主运行窗口
    ├── editor        # IDE代码编辑器
    |   ├── browser     # 代码编辑器核心
    |   ├── common      # 代码编辑器核心
    |   ├── contrib     # vscode 与独立 IDE共享的代码
    |   └── standalone  # 独立 IDE 独有的代码
    ├── platform      # 支持注入服务和平台相关基础服务（文件、剪切板、窗体、状态栏）
    ├── workbench     # 工作区UI布局，功能主界面
    │   ├── api              # 
    │   ├── browser          # 
    │   ├── common           # 
    │   ├── contrib          # 
    │   ├── electron-browser # 
    │   ├── services         # 
    │   └── test             # 
    ├── css.build.js  # 用于插件构建的CSS loader
    ├── css.js        # CSS loader
    ├── editor        # 对接IDE Core（读取编辑/交互状态），提供命令、上下文菜单、hover、snippet等支持
    ├── loader.js     # AMD loader（用于异步加载AMD模块）
    ├── nls.build.js  # 用于插件构建的NLS loader
    └── nls.js        # NLS（National Language Support）多语言loader
```
### 核心层
* base: 提供通用服务和构建用户界面
* platform: 注入服务和基础服务代码
* editor: 微软Monaco编辑器，也可独立运行使用
* wrokbench: 配合Monaco并且给viewlets提供框架：如：浏览器状态栏，菜单栏利用electron实现桌面程序

### 核心环境
整个项目完全使用typescript实现，electron中运行主进程和渲染进程，使用的api有所不同，所以在core中每个目录组织也是按照使用的api来安排，
运行的环境分为几类：
* common: 只使用javascritp api的代码，能在任何环境下运行
* browser: 浏览器api, 如操作dom; 可以调用common
* node: 需要使用node的api,比如文件io操作
* electron-brower: 渲染进程api, 可以调用common, brower, node, 依赖[electron renderer-process API](https://github.com/electron/electron/tree/master/docs#modules-for-the-renderer-process-web-page)
* electron-main: 主进程api, 可以调用: common, node 依赖于[electron main-process AP](https://github.com/electron/electron/tree/master/docs#modules-for-the-main-process)
