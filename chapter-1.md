# VSCode源码分析
## 简介
Visual Studio Code(简称VSCode) 是开源免费的IDE编辑器，原本是微软内部使用的云编辑器。

后来通过Eletron集成了桌面应用，并且可以跨平台使用，开发语言主要采用微软自家的TypeScript。
整个项目结构比较清晰，方便阅读理解。


## 编译安装
下载最新版本,目前我用的是1.37.1版本
官方的wiki中有编译安装的说明  [How to Contribute](https://github.com/microsoft/vscode/wiki/How-to-Contribute?_blank)

Linux, Window, MacOS三个系统编译时有些差别，参考官方文档，
在编译安装依赖时如果遇到connect timeout, 需要进行科学上网。

需要注意的一点 运行环境依赖版本 Nodejs x64 version >= 10.16.0, < 11.0.0,  python 2.7(3.0不能正常执行)

编译成功后会展示如下：

![avatar](https://camo.githubusercontent.com/48c7e4a793fdf74aca9b74011b5682196310d841/68747470733a2f2f692e696d6775722e636f6d2f443243655830792e706e67)
