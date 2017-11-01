provide/inject功能是vue2.2.0版本开始提供的，用于向子组件注入数据。

在时间顺序上，初始化inject要在初始化 method、data、computed、watch之前，而初始化provide要在初始化前几个字段之后：

```javascript
callHook(vm, 'beforeCreate')
// 初始化inject
initInjections(vm)
// 初始化method、data、computed、watch
initState(vm)
// 初始化provide
initProvide(vm)
callHook(vm, 'created')
```

provide是祖先组件的行为，它所做的只是把需要provide的属性挂到实例属性_provided上：

```javascript
function initProvide (vm: Component) {
  const provide = vm.$options.provide
  if (provide) {
    // 把provide的值挂到_provided，为了子组件去访问
    vm._provided = typeof provide === 'function'
      ? provide.call(vm)
      : provide
  }
}
```

祖先组件声明了provide的属性，后代组件就可以通过inject获取这些属性：

```javascript
export function resolveInject (inject: any, vm: Component): ?Object {
  if (inject) {
    // inject 可以是个数组，也可以是个对象
    // 对象形式是为了为inject的属性提供一个本地名称
    // 防止和其它字段名称冲突
    const isArray = Array.isArray(inject)
    const result = Object.create(null)
    const keys = isArray
      ? inject
      : hasSymbol
        ? Reflect.ownKeys(inject)
        : Object.keys(inject)

    for (let i = 0; i < keys.length; i++) {
      const key = keys[i]
      const provideKey = isArray ? key : inject[key]
      // 核心是沿着组件树向上查找节点的_provided是否有对应属性
      // 话说第一次查找不应该是vm.$parent吗？找自己的干啥。
      let source = vm
      while (source) {
        if (source._provided && provideKey in source._provided) {
          result[key] = source._provided[provideKey]
          break
        }
        source = source.$parent
      }
    }
    return result
  }
}
export function initInjections (vm: Component) {
  const result = resolveInject(vm.$options.inject, vm)
  if (result) {
    // 把inject的数据绑定到vm上
    // 把inject的数据变成响应式的
    // 然而只有provide的数据是响应式的时候inject数据的响应式才有意义
    Object.keys(result).forEach(key => {
      if (process.env.NODE_ENV !== 'production') {
        defineReactive(vm, key, result[key], () => {
          warn(
            `Avoid mutating an injected value directly since the changes will be ` +
            `overwritten whenever the provided component re-renders. ` +
            `injection being mutated: "${key}"`,
            vm
          )
        })
      } else {
        defineReactive(vm, key, result[key])
      }
    })
  }
}
```

虽然现在vue版本已经到了2.5了，我依然没见人用过这两个属性，介绍的文章也不多，[这篇文章提到一个简单地例子(请自备梯子)](https://medium.com/@znck/provide-inject-in-vue-2-2-b6473a7f7816)