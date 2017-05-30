data的初始化主要是利用了Observer类，在把数据变成响应式的时候也利用了Dep类。

```javascript
function initData (vm) {
  var data = vm.$options.data;
  // 将数据以```_data```挂载到vm上
  data = vm._data = typeof data === 'function'
    ? getData(data, vm)
    : data || {};

  // data必须是个对象
  if (!isPlainObject(data)) {
    data = {};
    "development" !== 'production' && warn(
      'data functions should return an object:\n' +
      'https://vuejs.org/v2/guide/components.html#data-Must-Be-a-Function',
      vm
    );
  }

  // 遍历data，并在vm上代理
  var keys = Object.keys(data);
  var props = vm.$options.props;
  var i = keys.length;
  while (i--) {
    if (props && hasOwn(props, keys[i])) {
      "development" !== 'production' && warn(
        "The data property \"" + (keys[i]) + "\" is already declared as a prop. " +
        "Use prop default value instead.",
        vm
      );
    } else if (!isReserved(keys[i])) {
      // 如果data的键名不是保留字，vm代理该属性
      proxy(vm, "_data", keys[i]);
    }
  }


  // 将数据变成响应式的
  observe(data, true /* asRootData */);
}
```

这里主要做了两件事：

* 将数据变成响应式的(就是之前observer那一套)
* 在vm上代理属性

这里需要解释的是如何让vm代理这些属性的:

```javasccript
proxy(vm, "_data", keys[i]);
function proxy (target, sourceKey, key) {
  sharedPropertyDefinition.get = function proxyGetter () {
    return this[sourceKey][key]
  };
  sharedPropertyDefinition.set = function proxySetter (val) {
    this[sourceKey][key] = val;
  };
  // 代理功能的关键代码
  Object.defineProperty(target, key, sharedPropertyDefinition);
}
```

当我们在某个属性字段的时候，我们实际上是访问的实例的```_data```下的对应字段，当然这些字段都变成响应式的了，我们会进入对应的getter。