
# VSCode源码分析 - 开发调试

## 目录
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-1.md">启动主进程</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-2.md">实例化服务</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-3.md">事件分发</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-4.md">进程通信</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-5.md">主要窗口</a>
* <a href="https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/chapter-6.md">开发调试</a>

## <a name="8">开发调试</a>

```js
app.once('ready', function () {
	//启动追踪
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
```
### 启动追踪
这里如果传入trace参数，在onReady启动之前会调用chromium的收集跟踪数据，
提供的底层的追踪工具允许我们深度了解 V8 的解析以及其他时间消耗情况，

一旦收到可以开始记录的请求，记录将会立马启动并且在子进程是异步记录听的. 当所有的子进程都收到 startRecording 请求的时候，callback 将会被调用.

categoryFilter是一个过滤器，它用来控制那些分类组应该被用来查找.过滤器应当有一个可选的 - 前缀来排除匹配的分类组.不允许同一个列表既是包含又是排斥.

#### contentTracing.startRecording(options, callback)
* options Object
	* categoryFilter String
	* traceOptions String
* callback Function

[关于trace的详细介绍]（https://www.w3cschool.cn/electronmanual/electronmanual-content-tracing.html）

### 结束追踪 
#### contentTracing.stopRecording(resultFilePath, callback)
* resultFilePath String
* callback Function
在成功启动窗口后，程序结束性能追踪，停止对所有子进程的记录.

子进程通常缓存查找数据，并且仅仅将数据截取和发送给主进程.这有利于在通过 IPC 发送查找数据之前减小查找时的运行开销，这样做很有价值.因此，发送查找数据，我们应当异步通知所有子进程来截取任何待查找的数据.

一旦所有子进程接收到了 stopRecording 请求，将调用 callback ，并且返回一个包含查找数据的文件.

如果 resultFilePath 不为空，那么将把查找数据写入其中，否则写入一个临时文件.实际文件路径如果不为空，则将调用 callback .

### debug
调试界面在菜单栏找到 Help->Toggle Developers Tools 

调出Chrome开发者调试工具进行调试
![img](https://github.com/fzxa/VSCode-sourcecode-analysis/blob/master/vscode-debugg.png)
