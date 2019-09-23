
# VSCode源码分析 - 实例化服务

## 目录
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">启动主进程</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-2.md">实例化服务</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-3.md">事件分发</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-4.md">进程通信</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-5.md">主要窗口</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-6.md">开发调试</a>


## <a name="4">实例化服务</a>

SyncDescriptor负责注册这些服务，当用到该服务时进程实例化使用

src/vs/platform/instantiation/common/descriptors.ts

```js
export class SyncDescriptor<T> {
	readonly ctor: any;
	readonly staticArguments: any[];
	readonly supportsDelayedInstantiation: boolean;
	constructor(ctor: new (...args: any[]) => T, staticArguments: any[] = [], supportsDelayedInstantiation: boolean = false) {
		this.ctor = ctor;
		this.staticArguments = staticArguments;
		this.supportsDelayedInstantiation = supportsDelayedInstantiation;
	}
}
```

main.ts中startup方法调用invokeFunction.get实例化服务
```js
await instantiationService.invokeFunction(async accessor => {
	const environmentService = accessor.get(IEnvironmentService);
	const configurationService = accessor.get(IConfigurationService);
	const stateService = accessor.get(IStateService);
	try {
		await this.initServices(environmentService, configurationService as ConfigurationService, stateService as StateService);
	} catch (error) {

		// Show a dialog for errors that can be resolved by the user
		this.handleStartupDataDirError(environmentService, error);

		throw error;
	}
});
```

get方法调用_getOrCreateServiceInstance，这里第一次创建会存入缓存中
下次实例化对象时会优先从缓存中获取对象。

src/vs/platform/instantiation/common/instantiationService.ts

```js
invokeFunction<R, TS extends any[] = []>(fn: (accessor: ServicesAccessor, ...args: TS) => R, ...args: TS): R {
	let _trace = Trace.traceInvocation(fn);
	let _done = false;
	try {
		const accessor: ServicesAccessor = {
			get: <T>(id: ServiceIdentifier<T>, isOptional?: typeof optional) => {

				if (_done) {
					throw illegalState('service accessor is only valid during the invocation of its target method');
				}

				const result = this._getOrCreateServiceInstance(id, _trace);
				if (!result && isOptional !== optional) {
					throw new Error(`[invokeFunction] unknown service '${id}'`);
				}
				return result;
			}
		};
		return fn.apply(undefined, [accessor, ...args]);
	} finally {
		_done = true;
		_trace.stop();
	}
}
private _getOrCreateServiceInstance<T>(id: ServiceIdentifier<T>, _trace: Trace): T {
	let thing = this._getServiceInstanceOrDescriptor(id);
	if (thing instanceof SyncDescriptor) {
		return this._createAndCacheServiceInstance(id, thing, _trace.branch(id, true));
	} else {
		_trace.branch(id, false);
		return thing;
	}
}
```
