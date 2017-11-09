在Vue中资源指的是component、filter和directive。全局注册资源是把资源挂载到Vue(或者Vue子类)的静态属性options上，在实例化的时候会把实例的资源属性与全局的资源进行合并(merge策略是通过Object.create继承全局资源)。

```javascript
const ASSET_TYPES = [
  'component',
  'directive',
  'filter'
]

  ASSET_TYPES.forEach(type => {
    // 全局注册的静态方法
    // 子类是直接引用这几个方法
    Vue[type] = function (
      id: string,
      definition: Function | Object
    ): Function | Object | void {
      // 只有资源id没有值时认为是获取全局资源
      if (!definition) {
        return this.options[type + 's'][id]
      } else {
        /* istanbul ignore if */
        if (process.env.NODE_ENV !== 'production') {
          if (type === 'component' && config.isReservedTag(id)) {
            warn(
              'Do not use built-in or reserved HTML elements as component ' +
              'id: ' + id
            )
          }
        }
        if (type === 'component' && isPlainObject(definition)) {
          definition.name = definition.name || id
          // 对于component，最终是转换成为了一个子类
          // 然而我一直想不通特意使用_base.extend而不是this.extend
          // 这样写组件注册树只有两层
          definition = this.options._base.extend(definition)
        }

        // 处理注册指令的一个语法糖
        if (type === 'directive' && typeof definition === 'function') {
          definition = { bind: definition, update: definition }
        }

        // 资源分类挂载到静态属性options上
        this.options[type + 's'][id] = definition
        return definition
      }
    }
  })

```