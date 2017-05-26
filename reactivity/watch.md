$watch这个方法是开放给开发者用来生成watcher实例的入口，主要做的就是实例化了一个watcher。

```javascript
Vue.prototype.$watch = function (expOrFn,cb,options) {
	var vm = this;
	options = options || {};
	// 标记这个watcher是用户行为生成的，需要以后容错
	options.user = true;
	var watcher = new Watcher(vm, expOrFn, cb, options);
	// 立即触发回调
	if (options.immediate) {
  		cb.call(vm, watcher.value);
	}

	// 返回取消观察函数，这个函数内部调用了watcher原型链上的teardown方法
	return function unwatchFn () {
		watcher.teardown();
	}
};
```