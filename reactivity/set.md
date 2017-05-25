vue实例上的```$set```方法所做的有以下几件事：

* 为对象添加属性
* 如果对象是响应式的，则新属性也要保证响应式
* 如果对象是响应式的，则要通过dep实例通知相关watcher


```javascript
function set (target, key, val) {
  // 数组通过splice方法曲线救国
  if (Array.isArray(target) && typeof key === 'number') {
    target.length = Math.max(target.length, key);
    target.splice(key, 1, val);
    return val
  }

  // 如果本身就有这个属性，直接走原来setter那条路就能实现通知和对新值观察
  if (hasOwn(target, key)) {
    target[key] = val;
    return val
  }
  var ob = (target ).__ob__;
  if (target._isVue || (ob && ob.vmCount)) {
    "development" !== 'production' && warn(
      'Avoid adding reactive properties to a Vue instance or its root $data ' +
      'at runtime - declare it upfront in the data option.'
    );
    return val
  }
  //  原始对象不是响应式的，为其添加新属性不用保证是响应式的
  if (!ob) {
    target[key] = val;
    return val
  }
  // 添加新属性，对新属性进行观察(响应式处理)
  defineReactive$$1(ob.value, key, val);
  // 通知watcher
  ob.dep.notify();
  return val
}
```

我在讲dep实例的时候，提到过一个对象，他可以有两个相关的dep实例，一个是视为其他对象的属性时的游离的dep实例，一个是在视为对象时和observer实例相关联的结合的dep实例。在依赖收集时(dep与watcher建立连接)两个dep实例都要和watcher建立联系。当时可能会奇怪为什么要有这个结合的dep实例，答案就在这里。当我们调用```$set```时需要通知相关的watcher，但是我们无法找到那个游离的dep实例，所以干脆建了一个新的dep实例。(讲到这我突然觉得哪里不对，把observer实例和游离的dep实例关联起来没什么难度啊，而且真的需要observer实例吗？)