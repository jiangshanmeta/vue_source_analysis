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

关于computed对应的watcher的选项有必要多说两句。```var computedWatcherOptions = { lazy: true };```。这个设定使得dep通知watcher数据变化时，watcher并不立即重新计算相应的值，而是先标记一下当前数据是脏数据，当我们真的需要这个计算属性的时候再去计算(下面代码中```evaluate```方法)。


```javascript
// 绑定到vm上
function defineComputed (target, key, userDef) {
  if (typeof userDef === 'function') {
    sharedPropertyDefinition.get = createComputedGetter(key);
    sharedPropertyDefinition.set = noop;
  } else {
    // 计算getter默认是从watcher中获取值
    // 通过设置cache属性控制getter是否强制重算
    sharedPropertyDefinition.get = userDef.get
      ? userDef.cache !== false
        ? createComputedGetter(key)
        : userDef.get
      : noop;

    // 允许设定计算setter
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
        // evaluate 方法是lazy watcher专用
        watcher.evaluate();
      }


      if (Dep.target) {
        watcher.depend();
      }

      // computed最终返回的是watcher的值
      return watcher.value
    }
  }
}
```

computed的每个属性绑定在vm上是通过```Object.defineProperty```实现的。

最后要说明的是```watcher.depend();```这句话是干啥的。在讲Watcher的时候，提到过一个嵌套的问题，还是以当时的例子为例，当我们调用完watcherB的evaluate方法后，watcherB的依赖收集已经完成，现在```Dep.target```上挂的是watcherA，watcherA需要和watcherB所依赖的所有dep实例建立关联，所以这里watcherB调用depend方法，使相应的dep实例与watcherA建立关联。