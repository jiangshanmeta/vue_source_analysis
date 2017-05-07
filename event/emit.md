触发自定义事件事件思路不复杂：根据事件名找到对应的回调数组，然后遍历执行就好了(注意this指向)。

```javascript
Vue.prototype.$emit = function (event) {
	var vm = this;
	{
		var lowerCaseEvent = event.toLowerCase();
		if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
			tip(
				"Event \"" + lowerCaseEvent + "\" is emitted in component " +
				(formatComponentName(vm)) + " but the handler is registered for \"" + event + "\". " +
				"Note that HTML attributes are case-insensitive and you cannot use " +
				"v-on to listen to camelCase events when using in-DOM templates. " +
				"You should probably use \"" + (hyphenate(event)) + "\" instead of \"" + event + "\"."
			);
		}
	}
	var cbs = vm._events[event];
	if (cbs) {
		cbs = cbs.length > 1 ? toArray(cbs) : cbs;
		var args = toArray(arguments, 1);
		for (var i = 0, l = cbs.length; i < l; i++) {
			cbs[i].apply(vm, args);
		}
	}
	return vm
};
```

这里有这么个问题，里面的```toArray```方法起到什么作用？我先把```toArray```的实现抄下来：

```javascript
function toArray (list, start) {
	start = start || 0;
	var i = list.length - start;
	var ret = new Array(i);
	while (i--) {
		ret[i] = list[i + start];
	}
	return ret
}
```

关于arguments的那个toArray，我想起来了underscore.js中```optimizeCb```这个函数，其本质上是arguments自身的性能问题。在这里将类数组arguments转换成真数组可以提高性能。

那针对回调数组的toArray起到了什么作用？联想到上一节所说的```$once```方法，我们可以看出回调在执行的时候可能会改变这个回调数组，如果我们根据原回调数组去遍历可能会出错。所以我们取了一个新回调数组，按照新回调数组去遍历，改变原回调数组并不会影响新回调数组。