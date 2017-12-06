vuex是vue官方出品的用来实现共享状态管理的一个插件。我之前也仿照vuex的API[实现了一个低配版](https://github.com/jiangshanmeta/metax)。实现vuex其实并不是太难，我的实现就五百行左右吧，官方实现多点但是核心也不超过一千行。实现vuex作为一个加深对vue理解的练手项目还是不错的。

vuex中的state和getters对应vue实例中的data和computed，vuex的mutations/commit、actions/dispatch本质上是事件机制，vuex的subscribe和subscribeAction更接近于钩子机制。可以说只要熟悉vue，实现vuex的前提条件就满足了。

我的实现和vuex最大的差异在于对module的理解。vuex的module更倾向于是提供一个业务代码组织方式，我的实现是把module当成一个独立单元(store实例)，这些store实例按照树状结构组织起来，并且相应的属性会按照命名空间被代理到根store实例上，最终暴露出来的是根store实例。vuex只会对应一个store实例，这一个store实例会对应一棵module实例树。在vuex中ModuleCollection负责连接store实例和module实例树，但是它更接近于一个工具类。

由module引申出来的一个概念是localContext。在vuex的文档中经常会看到rootState、rootGetters这样的字眼，他们其实对应rootContext。localContext是限制在了一个module中。在我的实现中，每一个module都会对应一个store实例，localContext可以认为是每个module对应的store实例。在vuex中由于只有一个store实例，需要做一些操作获取localContext。


```javascript
function makeLocalContext (store, namespace, path) {
  const noNamespace = namespace === ''

  const local = {
    // 对应dispatch方法第一个参数的dispatch
    dispatch: noNamespace ? store.dispatch : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        // 核心是根据命名空间获取相对于根的dispatch type参数
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._actions[type]) {
          console.error(`[vuex] unknown local action type: ${args.type}, global type: ${type}`)
          return
        }
      }

      return store.dispatch(type, payload)
    },
    // 对应dispatch方法第一个参数的commit
    // 思路与dispatch一致
    commit: noNamespace ? store.commit : (_type, _payload, _options) => {
      const args = unifyObjectStyle(_type, _payload, _options)
      const { payload, options } = args
      let { type } = args

      if (!options || !options.root) {
        type = namespace + type
        if (process.env.NODE_ENV !== 'production' && !store._mutations[type]) {
          console.error(`[vuex] unknown local mutation type: ${args.type}, global type: ${type}`)
          return
        }
      }

      store.commit(type, payload, options)
    }
  }

  // getters and state object must be gotten lazily
  // because they will be changed by vm update
  Object.defineProperties(local, {
    getters: {
      get: noNamespace
        ? () => store.getters
        : () => makeLocalGetters(store, namespace)
    },
    state: {
      get: () => getNestedState(store.state, path)
    }
  })

  return local
}

// 从store的getters中截取localContext需要的getter
function makeLocalGetters (store, namespace) {
  const gettersProxy = {}

  const splitPos = namespace.length
  Object.keys(store.getters).forEach(type => {
    // skip if the target getter is not match this namespace
    // 截取的依据是命名空间
    // 获取该module以及所有子module的getters
    if (type.slice(0, splitPos) !== namespace) return

    // extract local getter type
    const localType = type.slice(splitPos)

    // Add a port to the getters proxy.
    // Define as getter property because
    // we do not want to evaluate the getters in this time.
    Object.defineProperty(gettersProxy, localType, {
      get: () => store.getters[type],
      enumerable: true
    })
  })

  return gettersProxy
}

// 根据路径截取state
function getNestedState (state, path) {
  return path.length
    ? path.reduce((state, key) => state[key], state)
    : state
}
```

我个人认为比较难理解的是localGetters的实现。所有的getters都以hash的形式注册到了全局的getters上，要分辨属于哪个module只能通过module的命名空间和挂在全局getters上的键名来对比。在我的实现中获取localGetters比较容易，但是没考虑子module的getters。

关于module的还有注册各个属性，都比较好理解，唯一要说的是注册state：

```javascript
Vue.set(parentState, moduleName, module.state)
```

之所以用了Vue.set而不是简单地设置parentState，是因为考虑到parentState如果是响应式的，子module的state可以自动变成响应式的(registerModule的时候会有这种情况)。在registerModule的时候，我们会发现会重新生成一个vue实例，它的作用仅仅是更新getters，原来的state并不受这个新vue实例的影响(进一步说，旧有的依赖也不会因为这个新vue实例而受影响)。