# VSCode源码分析
## 简介
Visual Studio Code(简称VSCode) 是开源免费的IDE编辑器，原本是微软内部使用的云编辑器。

通过Eletron集成了桌面应用，可以跨平台使用，开发语言主要采用微软自家的TypeScript。
整个项目结构比较清晰，方便阅读理解。


## 编译安装
下载最新版本,目前我用的是1.37.1版本
官方的wiki中有编译安装的说明  [How to Contribute](https://github.com/microsoft/vscode/wiki/How-to-Contribute?_blank)

Linux, Window, MacOS三个系统编译时有些差别，参考官方文档，
在编译安装依赖时如果遇到connect timeout, 需要进行科学上网。

> 需要注意的一点 运行环境依赖版本 Nodejs x64 version >= 10.16.0, < 11.0.0,  python 2.7(3.0不能正常执行)

![avatar](https://camo.githubusercontent.com/48c7e4a793fdf74aca9b74011b5682196310d841/68747470733a2f2f692e696d6775722e636f6d2f443243655830792e706e67)

## 整体架构
![avatar](https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/vscode-source.png?raw=true)

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
├── typings       # 函数语法补全定义
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
    ├── loader.js     # AMD loader（用于异步加载AMD模块，类似于require.js）
    ├── nls.build.js  # 用于插件构建的NLS loader
    └── nls.js        # NLS（National Language Support）多语言loader
```

## 启动流程分析

### Eletron通过package.json中的main字段来定义应用入口。
main.js是vscode的入口。

```js
app.once('ready', function () {
	if (args['trace']) {
		// @ts-ignore
		const contentTracing = require('electron').contentTracing;

		const traceOptions = {
			categoryFilter: args['trace-category-filter'] || '*',
			traceOptions: args['trace-options'] || 'record-until-full,enable-sampling'
		};

		contentTracing.startRecording(traceOptions, () => onReady());
	} else {
		onReady();
	}
});
function onReady() {
	perf.mark('main:appReady');

	Promise.all([nodeCachedDataDir.ensureExists(), userDefinedLocale]).then(([cachedDataDir, locale]) => {
		//1. 这里尝试获取本地配置信息，如果有的话会传递到startup
		if (locale && !nlsConfiguration) {
			nlsConfiguration = lp.getNLSConfiguration(product.commit, userDataPath, metaDataFile, locale);
		}

		if (!nlsConfiguration) {
			nlsConfiguration = Promise.resolve(undefined);
		}

		
		nlsConfiguration.then(nlsConfig => {
			
			//4. 首先会检查用户语言环境配置，如果没有设置默认使用英语 
			const startup = nlsConfig => {
				nlsConfig._languagePackSupport = true;
				process.env['VSCODE_NLS_CONFIG'] = JSON.stringify(nlsConfig);
				process.env['VSCODE_NODE_CACHED_DATA_DIR'] = cachedDataDir || '';

				perf.mark('willLoadMainBundle');
				//使用微软的loader组件加载electron-main/main文件
				require('./bootstrap-amd').load('vs/code/electron-main/main', () => {
					perf.mark('didLoadMainBundle');
				});
			};

			// 2. 接收到有效的配置传入是其生效，调用startup
			if (nlsConfig) {
				startup(nlsConfig);
			}

			// 3. 这里尝试使用本地的应用程序
			// 应用程序设置区域在ready事件后才有效
			else {
				let appLocale = app.getLocale();
				if (!appLocale) {
					startup({ locale: 'en', availableLanguages: {} });
				} else {

					// 配置兼容大小写敏感，所以统一转换成小写
					appLocale = appLocale.toLowerCase();
					// 这里就会调用config服务，把本地配置加载进来再调用startup
					lp.getNLSConfiguration(product.commit, userDataPath, metaDataFile, appLocale).then(nlsConfig => {
						if (!nlsConfig) {
							nlsConfig = { locale: appLocale, availableLanguages: {} };
						}

						startup(nlsConfig);
					});
				}
			}
		});
	}, console.error);
}
```

### vs/code/electron-main/main.ts
electron-main/main 是程序真正启动的入口,进入main process初始化流程.
#### 这里主要做了两件事情：
1. 初始化Service 
2. 启动主实例

直接看startup方法的实现,基础服务初始化完成后会加载 CodeApplication, mainIpcServer, instanceEnvironment，调用 startup 方法启动APP
```js
private async startup(args: ParsedArgs): Promise<void> {

		//spdlog 日志服务
		const bufferLogService = new BufferLogService();
		
		// 1. 调用 createServices
		const [instantiationService, instanceEnvironment] = this.createServices(args, bufferLogService);
		try {

			// 1.1 初始化Service服务
			await instantiationService.invokeFunction(async accessor => {
				// 基础服务，包括一些用户数据，缓存目录
				const environmentService = accessor.get(IEnvironmentService);
				// 配置服务
				const configurationService = accessor.get(IConfigurationService);
				// 持久化数据
				const stateService = accessor.get(IStateService);

				try {
					await this.initServices(environmentService, configurationService as ConfigurationService, stateService as StateService);
				} catch (error) {

					// 抛出错误对话框
					this.handleStartupDataDirError(environmentService, error);

					throw error;
				}
			});

			// 1.2 启动实例
			await instantiationService.invokeFunction(async accessor => {
				const environmentService = accessor.get(IEnvironmentService);
				const logService = accessor.get(ILogService);
				const lifecycleService = accessor.get(ILifecycleService);
				const configurationService = accessor.get(IConfigurationService);

				const mainIpcServer = await this.doStartup(logService, environmentService, lifecycleService, instantiationService, true);

				bufferLogService.logger = new SpdLogService('main', environmentService.logsPath, bufferLogService.getLevel());
				once(lifecycleService.onWillShutdown)(() => (configurationService as ConfigurationService).dispose());
				
				return instantiationService.createInstance(CodeApplication, mainIpcServer, instanceEnvironment).startup();
			});
		} catch (error) {
			instantiationService.invokeFunction(this.quit, error);
		}
	}
```

这里通过createService创建一些基础的Service
```js
private createServices(args: ParsedArgs, bufferLogService: BufferLogService): [IInstantiationService, typeof process.env] {
	
	//服务注册容器
	const services = new ServiceCollection();

	const environmentService = new EnvironmentService(args, process.execPath);
	const instanceEnvironment = this.patchEnvironment(environmentService); // Patch `process.env` with the instance's environment
	services.set(IEnvironmentService, environmentService);

	const logService = new MultiplexLogService([new ConsoleLogMainService(getLogLevel(environmentService)), bufferLogService]);
	process.once('exit', () => logService.dispose());
	//日志服务
	services.set(ILogService, logService);
	//配置服务
	services.set(IConfigurationService, new ConfigurationService(environmentService.settingsResource));
	//生命周期
	services.set(ILifecycleService, new SyncDescriptor(LifecycleService));
	//状态存储
	services.set(IStateService, new SyncDescriptor(StateService));
	//网络请求
	services.set(IRequestService, new SyncDescriptor(RequestService));
	//主题设定
	services.set(IThemeMainService, new SyncDescriptor(ThemeMainService));
	//注册服务
	services.set(ISignService, new SyncDescriptor(SignService));

	return [new InstantiationService(services, true), instanceEnvironment];
}
```

#### vs/code/electron-main/app.ts
这里首先触发CodeApplication.startup()方法， 在第一个窗口打开3秒后成为共享进程，
```js
async startup(): Promise<void> {
	...

	// 1. 第一个窗口创建共享进程
	const sharedProcess = this.instantiationService.createInstance(SharedProcess, machineId, this.userEnv);
	const sharedProcessClient = sharedProcess.whenReady().then(() => connect(this.environmentService.sharedIPCHandle, 'main'));
	this.lifecycleService.when(LifecycleMainPhase.AfterWindowOpen).then(() => {
		this._register(new RunOnceScheduler(async () => {
			const userEnv = await getShellEnvironment(this.logService, this.environmentService);

			sharedProcess.spawn(userEnv);
		}, 3000)).schedule();
	});
	// 2. 创建app实例
	const appInstantiationService = await this.createServices(machineId, trueMachineId, sharedProcess, sharedProcessClient);


	// 3. 打开一个窗口 调用 openFirstWindow
	const windows = appInstantiationService.invokeFunction(accessor => this.openFirstWindow(accessor, electronIpcServer, sharedProcessClient));

	// 4. 窗口打开后执行生命周期和授权操作
	this.afterWindowOpen();
	...
}
```

openFirstWindow 主要实现
CodeApplication.openFirstWindow 首次开启窗口时，创建 Electron 的 IPC，使主进程和渲染进程间通信。
根据 environmentService 提供的相关参数调用windowsMainService.open 方法打开窗口
```js
private openFirstWindow(accessor: ServicesAccessor, electronIpcServer: ElectronIPCServer, sharedProcessClient: Promise<Client<string>>): ICodeWindow[] {

		...

		// Open our first window
		const macOpenFiles: string[] = (<any>global).macOpenFiles;
		const context = !!process.env['VSCODE_CLI'] ? OpenContext.CLI : OpenContext.DESKTOP;
		const hasCliArgs = hasArgs(args._);
		const hasFolderURIs = hasArgs(args['folder-uri']);
		const hasFileURIs = hasArgs(args['file-uri']);
		const noRecentEntry = args['skip-add-to-recently-opened'] === true;
		const waitMarkerFileURI = args.wait && args.waitMarkerFilePath ? URI.file(args.waitMarkerFilePath) : undefined;

		...

		// default: read paths from cli
		return windowsMainService.open({
			context,
			cli: args,
			forceNewWindow: args['new-window'] || (!hasCliArgs && args['unity-launch']),
			diffMode: args.diff,
			noRecentEntry,
			waitMarkerFileURI,
			gotoLineMode: args.goto,
			initialStartup: true
		});
	}
```
afterWindowOpen()
```js
private afterWindowOpen(): void {

	// Signal phase: after window open
	this.lifecycleService.phase = LifecycleMainPhase.AfterWindowOpen;

	// Remote Authorities
	this.handleRemoteAuthorities();
}
```
