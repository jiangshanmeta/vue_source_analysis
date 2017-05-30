watch这个功能的实现本身也是利用了Watcher类

```javascript
function initWatch (vm, watch) {
  for (var key in watch) {
    var handler = watch[key];
    if (Array.isArray(handler)) {
      // 观察一个属性，有多个回调
      for (var i = 0; i < handler.length; i++) {
        createWatcher(vm, key, handler[i]);
      }
    } else {
      createWatcher(vm, key, handler);
    }
  }
}

function createWatcher (vm, key, handler) {
  var options;
  // 允许传入options字段
  if (isPlainObject(handler)) {
    options = handler;
    handler = handler.handler;
  }

  // 回调可以是vm实例上的方法(一般而言是传入的method)
  if (typeof handler === 'string') {
    handler = vm[handler];
  }

  // 调用实例上的$watch，最终是实例化一个Watcher
  vm.$watch(key, handler, options);
}
```

了解了Watcher是如何实现的，这里就没什么难度了，只是对参数做了必要的预处理而已。