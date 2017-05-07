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

#### 数组化处理策略

这个策略考虑的是parent和child都有相关字段，而且都要进行保留，于是就把他们放在了一个数组中了。比如说各种钩子、watch字段。

```javascript
strats.watch = function (parentVal, childVal) {
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

#### data处理策略

