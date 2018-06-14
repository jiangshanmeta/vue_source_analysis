Vue2中引入了virtual dom用来对DOM节点进行描述，我们首先要关注的是如何生层vdom。

首先要解决的一个问题是，创建的vdom，描述的是那种节点。一个vdom可能是描述普通的DOM元素，也可能对应一个组件。

```javascript
// 下面的tag对应createElement第一个参数，表示节点类型


// 处理抽象组件component 由is属性决定对应的节点
if (isDef(data) && isDef(data.is)) {
    tag = data.is
}
if (!tag) {
// 处理tag是falsy值的情况，业务代码走到这基本就是bug了
    return createEmptyVNode()
}

// createElement第一个参数支持多种类型(字符串、对象、函数)
if (typeof tag === 'string') {
    let Ctor
    ns = (context.$vnode && context.$vnode.ns) || config.getTagNamespace(tag)
    if (config.isReservedTag(tag)) {
        // 普通DOM节点
        vnode = new VNode(
            config.parsePlatformTagName(tag), data, children,
                undefined, undefined, context
        )
    } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) {
        // 组件
        vnode = createComponent(Ctor, data, context, children, tag)
    } else {
        // 既不是普通DOM节点，也不是注册的组件，一般是bug了
        vnode = new VNode(
            tag, data, children,
            undefined, undefined, context
            )
    }
} else {
// 一般只有手写render函数才有可能走到这里
// tag应该是一个vue实例的配置项，或者是一个vue子类
// https://github.com/alexjoverm/v-runtime-template 用到了这个特性
    vnode = createComponent(tag, data, context, children)
}
```

我们注册组件的时候，可能注册的是vue选项，也可能是vue子类，还可能是异步加载函数，createComponent方法就负责对这些情况处理。

```javascript
if (isUndef(Ctor)) {
    return
}
// 这个是根Vue类
const baseCtor = context.$options._base

// 对象的情况，尝试处理成vue子类
if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor)
}

// 到现在应该都处理成函数的形式了
if (typeof Ctor !== 'function') {
    if (process.env.NODE_ENV !== 'production') {
        warn(`Invalid Component definition: ${String(Ctor)}`, context)
    }
    return
}

// 异步组件 具体内容见下一篇 异步组件
let asyncFactory
// 因为所有的Vue子类都会有cid作为唯一标示
// 没有cid意味着是异步组件
if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor, context)
    if (Ctor === undefined) {
        // 暂时用个空节点占位，如果没有loading component的话
        return createAsyncPlaceholder(
            asyncFactory,
            data,
            context,
            children,
            tag
        )
    }
}

// 生成对应的vnode
const name = Ctor.options.name || tag
const vnode = new VNode(
    `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
    data, undefined, undefined, undefined, context,
    { Ctor, propsData, listeners, tag, children },
    asyncFactory
)
```

到现在为止我们确定了vdom描述的是什么节点