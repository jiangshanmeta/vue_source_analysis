钩子这个概念本身没什么可以说的，毕竟已经见得多了。

之前讲参数预处理的时候，提到了钩子字段被处理成了一个数组，数组中每一项都是个函数(正常情况下)。说到这钩子调用思路就很明确了，遍历这个数组，每一个函数在Vue实例上调用即可。

```javascript
function callHook (vm, hook) {
  var handlers = vm.$options[hook];
  if (handlers) {
    for (var i = 0, j = handlers.length; i < j; i++) {
      try {
        handlers[i].call(vm);
      } catch (e) {
        handleError(e, vm, (hook + " hook"));
      }
    }
  }
  if (vm._hasHookEvent) {
    vm.$emit('hook:' + hook);
  }
}
```

在预处理的时候，我们对数组中的每一项并没有进行类型判断，并不能肯定每一项一定是函数，所以这里有```try catch```。