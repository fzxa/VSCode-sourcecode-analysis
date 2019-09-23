# VSCode源码分析 - 主要窗口
## 目录
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">简介</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">技术架构</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">启动主进程</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-2.md">实例化服务</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-3.md">事件分发</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-4.md">进程通信</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-5.md">主要窗口</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-6.md">开发调试</a>

## <a name="7">主要窗口</a>

workbench.ts中startup里面Workbench负责创建主界面
src/vs/workbench/browser/workbench.ts

```js
startup(): IInstantiationService {
	try {
		
		...

		instantiationService.invokeFunction(async accessor => {

			// 渲染主工作界面
			this.renderWorkbench(instantiationService, accessor.get(INotificationService) as NotificationService, storageService, configurationService);

			// 界面布局
			this.createWorkbenchLayout(instantiationService);

			// Layout
			this.layout();

			// Restore
			try {
				await this.restoreWorkbench(accessor.get(IEditorService), accessor.get(IEditorGroupsService), accessor.get(IViewletService), accessor.get(IPanelService), accessor.get(ILogService), lifecycleService);
			} catch (error) {
				onUnexpectedError(error);
			}
		});

		return instantiationService;
	} catch (error) {
		onUnexpectedError(error);

		throw error; // rethrow because this is a critical issue we cannot handle properly here
	}
}
```

渲染主工作台，渲染完之后加入到container中，container加入到parent, parent就是body了。

this.parent.appendChild(this.container);

```js
private renderWorkbench(instantiationService: IInstantiationService, notificationService: NotificationService, storageService: IStorageService, configurationService: IConfigurationService): void {

		...

		//TITLEBAR_PART 顶部操作栏
		//ACTIVITYBAR_PART 最左侧菜单选项卡
		//SIDEBAR_PART 左侧边栏，显示文件，结果展示等
		//EDITOR_PART 右侧窗口，代码编写,欢迎界面等
		//STATUSBAR_PART 底部状态栏
		[
			{ id: Parts.TITLEBAR_PART, role: 'contentinfo', classes: ['titlebar'] },
			{ id: Parts.ACTIVITYBAR_PART, role: 'navigation', classes: ['activitybar', this.state.sideBar.position === Position.LEFT ? 'left' : 'right'] },
			{ id: Parts.SIDEBAR_PART, role: 'complementary', classes: ['sidebar', this.state.sideBar.position === Position.LEFT ? 'left' : 'right'] },
			{ id: Parts.EDITOR_PART, role: 'main', classes: ['editor'], options: { restorePreviousState: this.state.editor.restoreEditors } },
			{ id: Parts.PANEL_PART, role: 'complementary', classes: ['panel', this.state.panel.position === Position.BOTTOM ? 'bottom' : 'right'] },
			{ id: Parts.STATUSBAR_PART, role: 'contentinfo', classes: ['statusbar'] }
		].forEach(({ id, role, classes, options }) => {
			const partContainer = this.createPart(id, role, classes);

			if (!configurationService.getValue('workbench.useExperimentalGridLayout')) {
				// TODO@Ben cleanup once moved to grid
				// Insert all workbench parts at the beginning. Issue #52531
				// This is primarily for the title bar to allow overriding -webkit-app-region
				this.container.insertBefore(partContainer, this.container.lastChild);
			}

			this.getPart(id).create(partContainer, options);
		});

		// 将工作台添加至container dom渲染
		this.parent.appendChild(this.container);
	}
```

workbench最后调用this.layout()方法，将窗口占据整个界面，渲染完成
```js
layout(options?: ILayoutOptions): void {
		if (!this.disposed) {
			this._dimension = getClientArea(this.parent);

			if (this.workbenchGrid instanceof Grid) {
				position(this.container, 0, 0, 0, 0, 'relative');
				size(this.container, this._dimension.width, this._dimension.height);

				// Layout the grid widget
				this.workbenchGrid.layout(this._dimension.width, this._dimension.height);
			} else {
				this.workbenchGrid.layout(options);
			}

			// Emit as event
			this._onLayout.fire(this._dimension);
		}
	}
```
