Vue.extend是用来创造Vue子类的。说是子类，但是一般都能完成特殊功能，后面要讲的组件其实就是特殊的Vue子类。一般而言很少需要我们自己调用Vue.exend方法，生成组件子类是自动完成的，[element-ui的message功能](https://jiangshanmeta.gitbooks.io/elementui_source_analysis/content/message.html)手动调用了Vue.extend方法创造了一个子类。

```javascript
Vue.extend = function (extendOptions: Object): Function {
  extendOptions = extendOptions || {}
  const Super = this
  // 父类id
  const SuperId = Super.cid

  // cachedCtors用来缓存创造的子类
  // 同一个extendOptions同一个父类只会生成一个子类
  const cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {})
  if (cachedCtors[SuperId]) {
    return cachedCtors[SuperId]
  }

  const name = extendOptions.name || Super.options.name
  if (process.env.NODE_ENV !== 'production') {
    if (!/^[a-zA-Z][\w-]*$/.test(name)) {
      warn(
        'Invalid component name: "' + name + '". Component names ' +
        'can only contain alphanumeric characters and the hyphen, ' +
        'and must start with a letter.'
      )
    }
  }

  // 子类构造函数
  const Sub = function VueComponent (options) {
    this._init(options)
  }

  // 继承父类的prototype
  Sub.prototype = Object.create(Super.prototype)

  // 子类的原型对象的constructor指回子类
  // 其实是屏蔽掉父类原型对象的constructor属性
  Sub.prototype.constructor = Sub

  // 子类id
  Sub.cid = cid++

  // 合并父类的options和子类的options
  Sub.options = mergeOptions(
    Super.options,
    extendOptions
  )
  Sub['super'] = Super


  // 定义的props代理到原型对象上
  // 实例初始化props时只需代理自身特有的props
  if (Sub.options.props) {
    initProps(Sub)
  }

  // 和上面类似，把公有的computed代理到原型对象上
  // 实例初始化computed时只需代理自身特有computed
  if (Sub.options.computed) {
    initComputed(Sub)
  }

  // 复制静态方法
  Sub.extend = Super.extend
  Sub.mixin = Super.mixin
  Sub.use = Super.use


  // 依然是静态方法的复制
  // component、directive、filter
  ASSET_TYPES.forEach(function (type) {
    Sub[type] = Super[type]
  })

  // 如果有名，把该子类当成一个组件注册
  if (name) {
    Sub.options.components[name] = Sub
  }

  Sub.superOptions = Super.options
  Sub.extendOptions = extendOptions
  Sub.sealedOptions = extend({}, Sub.options)

  // 缓存子类
  cachedCtors[SuperId] = Sub

  return Sub
}

// 公有的props代理到prototype上
function initProps (Comp) {
  const props = Comp.options.props
  for (const key in props) {
    proxy(Comp.prototype, `_props`, key)
  }
}

// 公有的computed的代理到prototype上
function initComputed (Comp) {
  const computed = Comp.options.computed
  for (const key in computed) {
    defineComputed(Comp.prototype, key, computed[key])
  }
}
```

如果熟悉javascript的继承上面代码就比较好理解了，静态方法就直接复制粘贴了，原定对象上继承使用了Object.create方法。还有要说的是关于props和computed，直接代理到了原型对象上，子类的实例在初始化时就不用重复代理了。