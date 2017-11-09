插件是按照特定规则编写的，用于完成某个特定功能的一组代码。

在Vue中，编写插件可能涉及以下形式：

* 添加指令
* 添加过滤器
* 添加组件
* 为Vue的原型添加方法
* 为Vue构造函数添加方法

有这么多可能的形式，Vue为其插件定义了如下编写规则：插件应当有一个公开方法```install```，```install```方法第一个参数是Vue构造函数。

```javascript
Vue.use = function (plugin) {
    // 防止多次注册
	if (plugin.installed) {
		return
	}
	
	// 允许其他参数，并数组化
	var args = toArray(arguments, 1);

	// 第一个参数是Vue构造函数
	// 对于子类来说是子类的构造函数，这样可以把插件作用域限定在子类，不污染全局
	args.unshift(this);

	// 调用插件安装函数，注册插件
	if (typeof plugin.install === 'function') {
		plugin.install.apply(plugin, args);
	} else if (typeof plugin === 'function') {
		plugin.apply(null, args);
	}

	// 标记插件已经注册
	plugin.installed = true;
	return this
};
```

Vue在在构造函数上暴露了一个```use```方法，用于注册插件。