$once方法监听一个自定义事件，但传入的回调只执行一次，触发后回调被移除。要实现这个功能，原始回调需要被包装一层形成新回调(下面代码中的on函数)，新回调除了调用原回调，还要负责触发后的移除工作(对应$off方法的调用)。


有了```$on```方法和```$off```方法，实现```$once```方法

```javascript
Vue.prototype.$once = function (event, fn) {
	var vm = this;
	function on () {
		vm.$off(event, on);
		fn.apply(vm, arguments);
	}
	on.fn = fn;
	vm.$on(event, on);
	return vm
};
```

上一节讲```$off```方法的实现时留了个问题，为什么遍历找回调是通过```cb === fn || cb.fn === fn```判断，原因就在这里。使用```$once```方法时，我们实际上注册的回调是```on```而不是```fn```，但是外界无法感知这个```on```方法，因而无法移除回调。为了解决这个问题，作者在新回调上添加了一个属性```fn```，这个属性指向原回调，通过新回调就可以找到原回调了。