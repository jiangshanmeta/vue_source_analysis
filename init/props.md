props属性一般是用在子实例上，但是Vue也允许应用到根实例上。与props配合使用的是propsData属性，这个属性似乎比较陌生，因为主要是Vue内部使用的。子实例要获取父实例传下来的值，就要通过propsData。

```javascript
// 初始化props
function initProps (vm: Component, propsOptions: Object) {
  const propsData = vm.$options.propsData || {}
  const props = vm._props = {}

  // 注释里说是为了更新props属性时能够通过数组快速迭代，才有这个_propKeys
  const keys = vm.$options._propKeys = []
  const isRoot = !vm.$parent

  // observerState.shouldConvert 是用来控制是否实例化Observer的
  // 对于子实例来说，如果某个props属性是对象/数组，对其进一步的observe是没有意义的
  // 对于根实例，一般而言没有props和propsData，有propsData的话需要观察
  observerState.shouldConvert = isRoot
  for (const key in propsOptions) {
    keys.push(key)

    // 验证propsData传下来的值是否合法
    // 必要时赋默认值
    const value = validateProp(key, propsOptions, propsData, vm)


    // 把props变成响应式的
    if (process.env.NODE_ENV !== 'production') {
      if (isReservedProp[key] || config.isReservedAttr(key)) {
        warn(
          `"${key}" is a reserved attribute and cannot be used as component prop.`,
          vm
        )
      }
      defineReactive(props, key, value, () => {
        if (vm.$parent && !observerState.isSettingProps) {
          warn(
            `Avoid mutating a prop directly since the value will be ` +
            `overwritten whenever the parent component re-renders. ` +
            `Instead, use a data or computed property based on the prop's ` +
            `value. Prop being mutated: "${key}"`,
            vm
          )
        }
      })
    } else {
      defineReactive(props, key, value)
    }

    // 类似于data，将props的每一个属性代理到实例上
    // 实现原理依然是Object.defineProperty 利用get set
    if (!(key in vm)) {
      proxy(vm, `_props`, key)
    }
  }
  // 后面要初始化的data需要观察，因此shouldConvert需要置为true
  observerState.shouldConvert = true
}
```

props初始化和data初始化非常类似，都包含把数据变为响应式的，代理到实例上，只是value来源不一样。props的value来自于父组件，相对data初始化来说就多一步校验value合法性。

```javascript
// 校验props合法性，返回合法的props值
export function validateProp (
  key: string,
  propOptions: Object,
  propsData: Object,
  vm?: Component
): any {
  const prop = propOptions[key]
  const absent = !hasOwn(propsData, key)

  // 从propsData获取父组件传递下来的值
  let value = propsData[key]

  // 布尔属性特殊处理
  // 类似于HTML5的布尔属性，vue的布尔属性也是只要声明置为true
  if (isType(Boolean, prop.type)) {
    if (absent && !hasOwn(prop, 'default')) {
      value = false
    } else if (!isType(String, prop.type) && (value === '' || value === hyphenate(key))) {
      value = true
    }
  }
  // 父实例没有传递下来值，采用默认值
  if (value === undefined) {
    value = getPropDefaultValue(vm, prop, key)
    // 默认值是新值，需要被观察
    const prevShouldConvert = observerState.shouldConvert
    observerState.shouldConvert = true
    observe(value)
    observerState.shouldConvert = prevShouldConvert
  }

  // 这一块是负责required和validator两个属性
  if (process.env.NODE_ENV !== 'production') {
    assertProp(prop, key, value, vm, absent)
  }
  return value
}
```