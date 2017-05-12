computed的实现就是利用了Watcher类。它仅有一个数据获取函数，而数据更新时的回调是空函数。

```javascript
function initComputed (vm, computed) {
  // 一个vue实例所有computed对应的watcher都保存在这
  var watchers = vm._computedWatchers = Object.create(null);

  for (var key in computed) {
    var userDef = computed[key];
    // 确定数据获取函数
    var getter = typeof userDef === 'function' ? userDef : userDef.get;
    {
      if (getter === undefined) {
        warn(
          ("No getter function has been defined for computed property \"" + key + "\"."),
          vm
        );
        getter = noop;
      }
    }
    // create internal watcher for the computed property.
    // 通过watcher实现computed功能
    watchers[key] = new Watcher(vm, getter, noop, computedWatcherOptions);

    if (!(key in vm)) {
      // 让vue实例代理这个computed属性
      defineComputed(vm, key, userDef);
    } else {
      if (key in vm.$data) {
        warn(("The computed property \"" + key + "\" is already defined in data."), vm);
      } else if (vm.$options.props && key in vm.$options.props) {
        warn(("The computed property \"" + key + "\" is already defined as a prop."), vm);
      }
    }
  }
}
```

关于computed对应的watcher的选项有必要多说两句。```var computedWatcherOptions = { lazy: true };```。这个设定使得dep通知watcher数据变化时，watcher并不立即重新计算相应的值，而是先标记一下当前数据有误，当我们真的需要这个计算属性的时候再去计算(下面代码中```evaluate```方法)。


```javascript
function defineComputed (target, key, userDef) {
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = createComputedGetter(key);
    sharedPropertyDefinition.set = noop;
  } else {
    // 这个cache选项是用来控制用缓存值还是重新计算新值
    sharedPropertyDefinition.get = userDef.get
      ? userDef.cache !== false
        ? createComputedGetter(key)
        : userDef.get
      : noop;
    sharedPropertyDefinition.set = userDef.set
      ? userDef.set
      : noop;
  }
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
function createComputedGetter (key) {
  return function computedGetter () {
    // 注意定义成getter/setter形式时的this指向
    var watcher = this._computedWatchers && this._computedWatchers[key];
    if (watcher) {
      // 标记数据有误，获取当前数据，否则直接返回缓存值
      if (watcher.dirty) {
        watcher.evaluate();
      }
      if (Dep.target) {
        watcher.depend();
      }

      return watcher.value
    }
  }
}
```

computed的每个属性绑定在vm上是通过```Object.defineProperty```实现的，类似的还有我们传入的data属性。