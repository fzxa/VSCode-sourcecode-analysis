## <a name="6">进程通信</a>

### 主进程
src/vs/code/electron-main/main.ts

main.ts在启动应用后就创建了一个主进程 main process，它可以通过electron中的一些模块直接与原生GUI交互。
```js
server = await serve(environmentService.mainIPCHandle);
once(lifecycleService.onWillShutdown)(() => server.dispose());
```
### 渲染进程
仅启动主进程并不能给你的应用创建应用窗口。窗口是通过main文件里的主进程调用叫BrowserWindow的模块创建的。
### 主进程与渲染进程之间的通信
在electron中，主进程与渲染进程有很多通信的方法。比如ipcRenderer和ipcMain，还可以在渲染进程使用remote模块。

### ipcMain & ipcRenderer
* 主进程：ipcMain
* 渲染进程：ipcRenderer

ipcMain模块和ipcRenderer是类EventEmitter的实例。

在主进程中使用ipcMain接收渲染线程发送过来的异步或同步消息，发送过来的消息将触发事件。

在渲染进程中使用ipcRenderer向主进程发送同步或异步消息，也可以接收到主进程的消息。

* 发送消息，事件名为 channel .
* 回应同步消息, 你可以设置 event.returnValue .
* 回应异步消息, 你可以使用 event.sender.send(...)

创建IPC服务
src/vs/base/parts/ipc/node/ipc.net.ts

这里返回一个promise对象，成功则createServer

```js
export function serve(hook: any): Promise<Server> {
	return new Promise<Server>((c, e) => {
		const server = createServer();

		server.on('error', e);
		server.listen(hook, () => {
			server.removeListener('error', e);
			c(new Server(server));
		});
	});
}
```

#### 创建信道

src/vs/code/electron-main/app.ts

* mainIpcServer
	* launchChannel
	
* electronIpcServer
	* updateChannel
	* issueChannel
	* workspacesChannel
	* windowsChannel
	* menubarChannel
	* urlChannel
	* storageChannel
	* logLevelChannel

```js
private openFirstWindow(accessor: ServicesAccessor, electronIpcServer: ElectronIPCServer, sharedProcessClient: Promise<Client<string>>): ICodeWindow[] {

		// Register more Main IPC services
		const launchService = accessor.get(ILaunchService);
		const launchChannel = new LaunchChannel(launchService);
		this.mainIpcServer.registerChannel('launch', launchChannel);

		// Register more Electron IPC services
		const updateService = accessor.get(IUpdateService);
		const updateChannel = new UpdateChannel(updateService);
		electronIpcServer.registerChannel('update', updateChannel);

		const issueService = accessor.get(IIssueService);
		const issueChannel = new IssueChannel(issueService);
		electronIpcServer.registerChannel('issue', issueChannel);

		const workspacesService = accessor.get(IWorkspacesMainService);
		const workspacesChannel = new WorkspacesChannel(workspacesService);
		electronIpcServer.registerChannel('workspaces', workspacesChannel);

		const windowsService = accessor.get(IWindowsService);
		const windowsChannel = new WindowsChannel(windowsService);
		electronIpcServer.registerChannel('windows', windowsChannel);
		sharedProcessClient.then(client => client.registerChannel('windows', windowsChannel));

		const menubarService = accessor.get(IMenubarService);
		const menubarChannel = new MenubarChannel(menubarService);
		electronIpcServer.registerChannel('menubar', menubarChannel);

		const urlService = accessor.get(IURLService);
		const urlChannel = new URLServiceChannel(urlService);
		electronIpcServer.registerChannel('url', urlChannel);

		const storageMainService = accessor.get(IStorageMainService);
		const storageChannel = this._register(new GlobalStorageDatabaseChannel(this.logService, storageMainService));
		electronIpcServer.registerChannel('storage', storageChannel);

		// Log level management
		const logLevelChannel = new LogLevelSetterChannel(accessor.get(ILogService));
		electronIpcServer.registerChannel('loglevel', logLevelChannel);
		sharedProcessClient.then(client => client.registerChannel('loglevel', logLevelChannel));
		
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

每一个信道，内部实现两个方法 listen和call

例如：src/vs/platform/localizations/node/localizationsIpc.ts

构造函数绑定事件
```js
export class LocalizationsChannel implements IServerChannel {

	onDidLanguagesChange: Event<void>;

	constructor(private service: ILocalizationsService) {
		this.onDidLanguagesChange = Event.buffer(service.onDidLanguagesChange, true);
	}

	listen(_: unknown, event: string): Event<any> {
		switch (event) {
			case 'onDidLanguagesChange': return this.onDidLanguagesChange;
		}

		throw new Error(`Event not found: ${event}`);
	}

	call(_: unknown, command: string, arg?: any): Promise<any> {
		switch (command) {
			case 'getLanguageIds': return this.service.getLanguageIds(arg);
		}

		throw new Error(`Call not found: ${command}`);
	}
}


```
