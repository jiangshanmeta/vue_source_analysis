当获取一个vue实例的时候，首先要做的是对传入的参数做一些预处理，参数不同字段的处理方法是不同的，这些不同的处理策略挂到了```strats```这个对象上。需要提前说明的是有些字段是预置的，所以下面代码中会有```parent```和```child```，前者是预置的，后者是我们传入的。

每一个处理方法都会按序传入下面四个参数：parent字段值、child字段值、vm Vue实例、要处理的字段。


#### 默认处理策略

```javascript
var defaultStrat = function (parentVal, childVal) {
  return childVal === undefined
    ? parentVal
    : childVal
};
```

上面是默认的处理方法，代码上没有什么太难理解的。比如```el```字段，这个字段没有预置，只有我们自己传递的，最终是原样返回了传入的```el```字段。

#### 钩子处理策略

vue提供了诸如```beforeCreate```、```created```这样的钩子，这些钩子的预处理处理方式都是一致的。

```javascript
function mergeHook (
  parentVal,
  childVal
) {
  return childVal
    ? parentVal
      ? parentVal.concat(childVal)
      : Array.isArray(childVal)
        ? childVal
        : [childVal]
    : parentVal
}
```

最终的结果是得到一个数组，数组中的每个元素是个function。

#### 