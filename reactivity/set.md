由于Vue的实现机制，为一个对象添加新属性是无法被监听到，```$set```这个API设计出来就是解决这个问题的。

```javascript
function set (target, key, val) {
  // 对数组属性设定，走重载的变异方法
  if (Array.isArray(target) && typeof key === 'number') {
    target.length = Math.max(target.length, key);
    target.splice(key, 1, val);
    return val
  }
  // 已有属性，走setter那一套即可
  if (hasOwn(target, key)) {
    target[key] = val;
    return val
  }
  // 数据相关的那个observer
  var ob = (target ).__ob__;

  // 不能为vue实例和根数据对象添加属性
  if (target._isVue || (ob && ob.vmCount)) {
    "development" !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    );
    return val
  }

  // 对象非响应式，直接赋值
  if (!ob) {
    target[key] = val;
    return val
  }

  // 为对象添加新属性，让新属性成为响应式的
  defineReactive$$1(ob.value, key, val);

  // 通知相关watcher这个对象数据更新
  ob.dep.notify();
  return val
}
```

真正的核心代码是这一句：```defineReactive$$1(ob.value, key, val);```，这一句为对象添加了新的属性，并且使得值为响应式的。