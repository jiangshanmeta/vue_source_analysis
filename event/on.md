$on方法类似于```addEventListener```，用于监听某个事件，添加事件触发时执行的回调。

```javascript
Vue.prototype.$on = function (event, fn) {
	var this$1 = this;
	var vm = this;
	if (Array.isArray(event)) {
		for (var i = 0, l = event.length; i < l; i++) {
			this$1.$on(event[i], fn);
		}
	} else {
	  (vm._events[event] || (vm._events[event] = [])).push(fn);
		if (hookRE.test(event)) {
			vm._hasHookEvent = true;
		}
	}
	return vm
};
```

最核心的代码是这一行：```(vm._events[event] || (vm._events[event] = [])).push(fn);```。Vue把所有的自定义事件都组织在了```_event```下，每个事件维护一个数组，数组的每一项是触发事件时执行的回调。

作者还考虑到了多个事件有相同回调的情况，所以就有了```Array.isArray(event)```那个判断。但我个人认为这个设计不是很合理。


```javascript
Vue.prototype.$on = function (event, fn) {
	var this$1 = this;

	var vm = this;
	if (typeof event === 'object') {
		for(key in event){
			this$1.$on(key,event[key]);
		}
	} else {
	  (vm._events[event] || (vm._events[event] = [])).push(fn);
		if (hookRE.test(event)) {
			vm._hasHookEvent = true;
		}
	}
	return vm
};
```

我考虑的是传入一个对象，对象的键是事件名，值是回调。jQuery中经常看到允许这样传参的方法。