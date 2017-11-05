初始化这一章主要包括以下内容：我们传入的参数(provide/inject/method/data/computed/watch)是如何被预处理的、如何初始化。

与此相关的几个属性是：$options、$data、$props。我们对传入的数据合并后都是挂在$options下，包括我们自定义的属性。$data与$props分别反映了data和props，但是本质只是一层代理：

```javascript
  const dataDef = {}
  // 访问$data实质上是访问_data
  dataDef.get = function () { return this._data }
  const propsDef = {}
  // 访问$props实际是访问_props
  propsDef.get = function () { return this._props }
  if (process.env.NODE_ENV !== 'production') {
    dataDef.set = function (newData: Object) {
      warn(
        'Avoid replacing instance root $data. ' +
        'Use nested data properties instead.',
        this
      )
    }
    propsDef.set = function () {
      warn(`$props is readonly.`, this)
    }
  }
  // 注意是定义到原型对象上的而不是定义在实例上
  // 这样只需定义一次
  // 类似的操作还有Vue.extend创建子类时代理公有的computed和props
  Object.defineProperty(Vue.prototype, '$data', dataDef)
  Object.defineProperty(Vue.prototype, '$props', propsDef)
```