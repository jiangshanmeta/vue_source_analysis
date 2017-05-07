这个API根据参数的不同实现了以下几个功能：

* 取消监听所有事件
* 取消监听某一个事件
* 移除某个事件的某个回调
* 移除多个事件的某个回调

既然已经知道了这个自定义事件的数据组织形式，实现这些功能并不复杂：

* 取消监听所有事件，把```_event```清空
* 取消监听某个事件，把```_event[eventName]```清空
* 移除某个事件的某个回调，遍历一遍找到回调然后从数组中移除
* 移除多个事件的某个回调，利用第三个功能即可

```javascript
Vue.prototype.$off = function (event, fn) {
	var this$1 = this;
	var vm = this;
	//取消监听所有事件
	if (!arguments.length) {
		vm._events = Object.create(null);
		return vm
	}

	// 取消监听多个事件的特定回调
	if (Array.isArray(event)) {
		for (var i$1 = 0, l = event.length; i$1 < l; i$1++) {
			this$1.$off(event[i$1], fn);
		}
		return vm
	}
	var cbs = vm._events[event];
	if (!cbs) {
		return vm
	}

	// 取消监听某个事件
	if (arguments.length === 1) {
		vm._events[event] = null;
		return vm
	}
	var cb;
	var i = cbs.length;
	// 取消监听某个事件的特定回调
	while (i--) {
		cb = cbs[i];
		if (cb === fn || cb.fn === fn) {
			cbs.splice(i, 1);
			break
		}
	}
	return vm
};
```

根据上面所拆解的功能，这段代码就很好理解了，我也添加了必要的注释。

还有几点小细节：

作者在这里实现了链式调用，不过这也是标配了吧。

实现第三个功能时的遍历，作者采用了while循环逆向查找，这么做是为了减少操作优化性能，具体原理请看《高性能JavaScript》。

判断是否是特定回调时，使用```cb === fn || cb.fn === fn```，你可能会奇怪为什么有```cb.fn === fn```这个判断，看了下面的```$once```方法的实现你就明白了。