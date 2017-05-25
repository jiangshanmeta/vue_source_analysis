vue实例上的```$delete```方法是一个使用频率很低的方法，它有如下功能：

* 删除对象上的某个属性
* 通知watcher实例对象状态的改变

```javascript
function del (target, key) {
  // 数组的通过splice方法删除元素和实现通知
  if (Array.isArray(target) && typeof key === 'number') {
    target.splice(key, 1);
    return
  }
  var ob = (target ).__ob__;
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

  // 对象的状态更新通知
  ob.dep.notify();
}
```

这个API的具体实现和```$set```方法类似，不做过多介绍了。