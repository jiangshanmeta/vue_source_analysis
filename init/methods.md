经过strat提供的策略预处理之后，hash结构的methods以methods为标识符挂载到了vue实例的$options上，但是我们使用method的时候method是挂载在vue实例上的，下面的代码就是对应的实现

```javascript
function initMethods (vm, methods) {
  var props = vm.$options.props;
  for (var key in methods) {
    vm[key] = methods[key] == null ? noop : bind(methods[key], vm);
    {
      if (methods[key] == null) {
        warn(
          "method \"" + key + "\" has an undefined value in the component definition. " +
          "Did you reference the function correctly?",
          vm
        );
      }
      if (props && hasOwn(props, key)) {
        warn(
          ("method \"" + key + "\" has already been defined as a prop."),
          vm
        );
      }
    }
  }
}
```

要vue实例代理这些方法，只需要遍历赋值就好了。作者这里的```bind```方法不是Function原型上的bind方法，而是自行实现的，只处理了this。但是这里真的需要绑定this吗？我个人在使用的时候一般是以```this.methodName()```或者```vm.methodName()```的形式，这时候this的指向就是vm啊。我猜作者是考虑有人会这么用：```var mth = this.methodName;   mth();```。



至于这里的```props```暂不涉及。

