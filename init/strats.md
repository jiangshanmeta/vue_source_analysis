当获取一个vue实例的时候，首先要做的是对传入的参数做一些预处理，参数不同字段的处理方法是不同的，这些不同的处理策略挂到了```strats```这个对象上。需要提前说明的是有些字段是预置的，所以下面代码中会有```parent```和```child```，前者是预置的，后者是我们传入的。

每一个处理方法都会按序传入下面四个参数：parent字段值、child字段值、vm Vue实例、要处理的字段。


#### 默认处理策略

默认处理策略的规则是如果有child就返回child，否则返回parent。

```javascript
var defaultStrat = function (parentVal, childVal) {
  return childVal === undefined
    ? parentVal
    : childVal
};
```

这一个策略是兜底处理策略，源码中只提到el字段采用这一策略，其实template、render以及我们自定义传入的字段都会默认采用这一策略(自定义字段可以[自定义处理策略](https://cn.vuejs.org/v2/api/#optionMergeStrategies))。


#### 数组化处理策略

这个策略考虑的是parent和child都有相关字段，而且都要进行保留，于是就把他们放在了一个数组中了。比如说各种钩子、watch字段。

```javascript
// watch处理策略
strats.watch = function (parentVal, childVal) {
  // 其实watch的数组化处理是不完全的
  // 所以在初始化watch的时候会有判断是不是数组
  if (!childVal) { return Object.create(parentVal || null) }
  if (!parentVal) { return childVal }
  var ret = {};
  extend(ret, parentVal);
  for (var key in childVal) {
    var parent = ret[key];
    var child = childVal[key];
    if (parent && !Array.isArray(parent)) {
      parent = [parent];
    }
    ret[key] = parent
      ? parent.concat(child)
      : [child];
  }
  return ret
};

// 钩子处理策略
function mergeHook (
  parentVal: ?Array<Function>,
  childVal: ?Function | ?Array<Function>
): ?Array<Function> {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}
```

#### 覆盖处理策略

同样是parent和child都有相关字段，这个处理方式考虑的是抛弃掉parent的相关字段，采用child的字段。这个处理方式适用于methods、computed和props。

```javascript
strats.props =
strats.methods =
strats.computed = function (parentVal, childVal) {
  if (!childVal) { return Object.create(parentVal || null) }
  if (!parentVal) { return childVal }
  var ret = Object.create(null);
  extend(ret, parentVal);
  extend(ret, childVal);
  return ret
};
```

采用同样思路的还有对资源(component、directive、filter)的处理上：

```javascript
function mergeAssets (parentVal: ?Object, childVal: ?Object): Object {
  const res = Object.create(parentVal || null)
  return childVal
    ? extend(res, childVal)
    : res
}
```

#### data处理策略

data的处理相对比较复杂，主要是牵扯到是在实例上调用合并还是在创造子类(Vue.extend)时调用合并。

```javascript
strats.data = function (
  parentVal: any,
  childVal: any,
  vm?: Component
): ?Function {
  // Vue.extend 时的合并策略
  if (!vm) {

    if (!childVal) {
      return parentVal
    }

    // child类型必须为function
    // 因为考虑到引用类型的问题
    if (typeof childVal !== 'function') {
      process.env.NODE_ENV !== 'production' && warn(
        'The "data" option should be a function ' +
        'that returns a per-instance value in component ' +
        'definitions.',
        vm
      )
      return parentVal
    }
    // 直接Vue.extend是没有parentVal的
    // Vue子类调用extend方法才有parentVal
    // 因为是子类，其data应为function类型(如果有的话)
    if (!parentVal) {
      return childVal
    }

    // 到这里是Vue的子类调用extend方法
    // 而且都有data值(都是函数类型)
    // Vue的extend方法返回的是新子类，data应为function类型
    return function mergedDataFn () {
      // 把data值合并
      return mergeData(
        childVal.call(this),
        parentVal.call(this)
      )
    }
  } else if (parentVal || childVal) {
  // 实例上合并data
    return function mergedInstanceDataFn () {
      // 子实例的data
      const instanceData = typeof childVal === 'function'
        ? childVal.call(vm)
        : childVal
      // 父类的data
      const defaultData = typeof parentVal === 'function'
        ? parentVal.call(vm)
        : undefined
      if (instanceData) {
        return mergeData(instanceData, defaultData)
      } else {
        return defaultData
      }
    }
  }
}
```

data合并策略还有一点要注意的是最终返回的是一个函数，这个函数调用才返回合并后的data。