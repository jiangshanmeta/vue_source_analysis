一般定义组件的方式是使用一个对象，对于异步组件，使用一个工厂函数定义组件(区分是否是异步组件的依据)。

异步组件的基本思路是异步加载组件，当加载完成后通知对应的vue实例更新状态，vue实例重新解析要渲染的组件。加载期间可以渲染一个加载组件，默认是什么都不渲染。

下面的代码取自 *resolve-async-component.js* 文件

```javascript
// factory 是工厂函数
// baseCtor 是Vue，用来将组件对象加工成子类
// context 是使用这个组件的vue实例

export function resolveAsyncComponent (
  factory: Function,
  baseCtor: Class<Component>,
  context: Component
): Class<Component> | void {
  // error是加载失败的标志位
  // errorComp 是失败时使用的组件对应的vue子类
  if (isTrue(factory.error) && isDef(factory.errorComp)) {
    return factory.errorComp
  }

  // 如果加载成功，返回异步组件对应的子类
  if (isDef(factory.resolved)) {
    return factory.resolved
  }

  // 加载时展示的组件
  if (isTrue(factory.loading) && isDef(factory.loadingComp)) {
    return factory.loadingComp
  }

  // contexts 变量保存的是使用这个工厂函数的vue实例
  if (isDef(factory.contexts)) {
    // already pending
    factory.contexts.push(context)
  } else {
    const contexts = factory.contexts = [context]
    let sync = true

    // 加载成功或者失败都调用这个方法更新
    const forceRender = () => {
      for (let i = 0, l = contexts.length; i < l; i++) {
        // vue实例刷新
        contexts[i].$forceUpdate()
      }
    }

    // 加载成功
    const resolve = once((res: Object | Class<Component>) => {
      // 缓存异步组件对应的vue子类
      factory.resolved = ensureCtor(res, baseCtor)
      // 如果是同步的会在最后返回这个假装是异步的同步组件，不需要更新
      if (!sync) {
        forceRender()
      }
    })

    // 加载失败
    const reject = once(reason => {
      process.env.NODE_ENV !== 'production' && warn(
        `Failed to resolve async component: ${String(factory)}` +
        (reason ? `\nReason: ${reason}` : '')
      )
      if (isDef(factory.errorComp)) {
        // 标记为加载失败
        factory.error = true
        forceRender()
      }
    })

    const res = factory(resolve, reject)

    // 对不同语法的支持，默认是require语法，不做特殊处理

    if (isObject(res)) {

      if (typeof res.then === 'function') {
        // ()=>import 语法
        if (isUndef(factory.resolved)) {
          res.then(resolve, reject)
        }
      } else if (isDef(res.component) && typeof res.component.then === 'function') {
        // 处理加载状态语法

        res.component.then(resolve, reject)

        if (isDef(res.error)) {
          factory.errorComp = ensureCtor(res.error, baseCtor)
        }

        if (isDef(res.loading)) {
          factory.loadingComp = ensureCtor(res.loading, baseCtor)
          // 其实我没怎么理解这个delay是为什么场景设计的
          if (res.delay === 0) {
            factory.loading = true
          } else {
            setTimeout(() => {
              if (isUndef(factory.resolved) && isUndef(factory.error)) {
                factory.loading = true
                forceRender()
              }
            }, res.delay || 200)
          }
        }

        // 加载超时机制
        if (isDef(res.timeout)) {
          setTimeout(() => {
            if (isUndef(factory.resolved)) {
              reject(
                process.env.NODE_ENV !== 'production'
                  ? `timeout (${res.timeout}ms)`
                  : null
              )
            }
          }, res.timeout)
        }
      }
    }

    sync = false
    // return in case resolved synchronously
    return factory.loading
      ? factory.loadingComp
      : factory.resolved
  }
}

```