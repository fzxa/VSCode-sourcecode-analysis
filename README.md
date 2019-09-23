
# VSCode源码分析

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
