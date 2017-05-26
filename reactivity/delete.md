由于Vue的实现机制，删除对象的某个属性是无法被监听到的，```$delete```这个API就是设计出来解决这个问题的。

```javascript
function del (target, key) {
  // 删除数组元素，走重载的变异方法
  if (Array.isArray(target) && typeof key === 'number') {
    target.splice(key, 1);
    return
  }
  var ob = (target ).__ob__;
  // 不能删除vue实例或者跟数据元素上的属性
  if (target._isVue || (ob && ob.vmCount)) {
    "development" !== 'production' && warn(
      'Avoid deleting properties on a Vue instance or its root $data ' +
      '- just set it to null.'
    );
    return
  }
  if (!hasOwn(target, key)) {
    return
  }

  // 对象的删除元素
  delete target[key];
  if (!ob) {
    return
  }

  // 通知watcher数据更新
  ob.dep.notify();
}
```