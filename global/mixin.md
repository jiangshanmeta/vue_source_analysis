全局mixin的实现非常简单，就是把旧有的options和选项merge构成新的options：

```javascript
Vue.mixin = function (mixin: Object) {
    this.options = mergeOptions(this.options, mixin)
}
```

但是这个操作非常危险，一方面是因为它是全局性的，另一方面是因为它会重写类的静态属性options。在实例化的时候，会比较子类缓存的父类options和父类options是否一致：

```javascript
function resolveConstructorOptions (Ctor: Class<Component>) {
  let options = Ctor.options
  // Vue创建子类时通过静态属性super引用父类
  if (Ctor.super) {
    const superOptions = resolveConstructorOptions(Ctor.super)
    const cachedSuperOptions = Ctor.superOptions
    // 子类缓存的父类options与父类现有options对比
    // options被重写只有使用全局mixin的时候
    if (superOptions !== cachedSuperOptions) {
      Ctor.superOptions = superOptions

      // 子类也可能调用mixin全局注册
      // 相当于创建子类时使用mixin选项
      // 因此把他们合并到extendOptions中
      const modifiedOptions = resolveModifiedOptions(Ctor)
      if (modifiedOptions) {
        extend(Ctor.extendOptions, modifiedOptions)
      }
      // 重新合并构成新的子类options
      options = Ctor.options = mergeOptions(superOptions, Ctor.extendOptions)
      if (options.name) {
        options.components[options.name] = Ctor
      }
    }
  }
  return options
}
function resolveModifiedOptions (Ctor: Class<Component>): ?Object {
  let modified
  const latest = Ctor.options
  const extended = Ctor.extendOptions
  // sealedOptions是生成子类时的子类options快照
  const sealed = Ctor.sealedOptions
  for (const key in latest) {
    // 当前值和快照值不一致，说明子类调用mixin了
    if (latest[key] !== sealed[key]) {
      if (!modified) modified = {}
      modified[key] = dedupe(latest[key], extended[key], sealed[key])
    }
  }
  return modified
}
function dedupe (latest, extended, sealed) {
  // 值为array类型，是钩子或者watch，原注释只提到了钩子
  if (Array.isArray(latest)) {
    const res = []
    sealed = Array.isArray(sealed) ? sealed : [sealed]
    extended = Array.isArray(extended) ? extended : [extended]
    for (let i = 0; i < latest.length; i++) {
      // 保留生成子类时选项拥有的和子类通过mixin全局注册的
      // 把从父类选项合并的干掉，因为后面还会和父类的选项合并一遍
      if (extended.indexOf(latest[i]) >= 0 || sealed.indexOf(latest[i]) < 0) {
        res.push(latest[i])
      }
    }
    return res
  } else {
    return latest
  }
}
```

当父类options改变时，子类在实例化时需要重新确定自身的options。所以在Vue中使用全局mixin，应尽量保证在创造Vue子类之前(Vue.component、Vue.extend)。