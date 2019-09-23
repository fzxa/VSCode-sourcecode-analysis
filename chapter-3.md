# VSCode源码分析 - 事件分发
## 目录
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">简介</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">技术架构</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">启动主进程</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-2.md">实例化服务</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-3.md">事件分发</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-4.md">进程通信</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-5.md">主要窗口</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-6.md">开发调试</a>

## <a name="5">事件分发</a> 
### event 
src/vs/base/common/event.ts

程序中常见使用once方法进行事件绑定, 给定一个事件，返回一个只触发一次的事件，放在匿名函数返回
```js
export function once<T>(event: Event<T>): Event<T> {
	return (listener, thisArgs = null, disposables?) => {
		// 设置次变量，防止事件重复触发造成事件污染
		let didFire = false;
		let result: IDisposable;
		result = event(e => {
			if (didFire) {
				return;
			} else if (result) {
				result.dispose();
			} else {
				didFire = true;
			}

			return listener.call(thisArgs, e);
		}, null, disposables);

		if (didFire) {
			result.dispose();
		}

		return result;
	};
}
```
循环派发了所有注册的事件， 事件会存储到一个事件队列，通过fire方法触发事件

private _deliveryQueue?: LinkedList<[Listener<T>, T]>;//事件存储队列

```js
fire(event: T): void {
	if (this._listeners) {
		// 将所有事件传入 delivery queue
		// 内部/嵌套方式通过emit发出.
		// this调用事件驱动

		if (!this._deliveryQueue) {
			this._deliveryQueue = new LinkedList();
		}

		for (let iter = this._listeners.iterator(), e = iter.next(); !e.done; e = iter.next()) {
			this._deliveryQueue.push([e.value, event]);
		}

		while (this._deliveryQueue.size > 0) {
			const [listener, event] = this._deliveryQueue.shift()!;
			try {
				if (typeof listener === 'function') {
					listener.call(undefined, event);
				} else {
					listener[0].call(listener[1], event);
				}
			} catch (e) {
				onUnexpectedError(e);
			}
		}
	}
}
```
