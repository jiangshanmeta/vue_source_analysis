对于一个对象，observer通过```walk```方法去观察每一个属性。


```javascript
Observer.prototype.walk = function walk (obj) {
  var keys = Object.keys(obj);
  for (var i = 0; i < keys.length; i++) {
    defineReactive$$1(obj, keys[i], obj[keys[i]]);
  }
};
```

实现观察一个对象的每个属性的是```defineReactive$$1```方法


```javascript
function defineReactive$$1 (obj,key,val,customSetter) {
  var dep = new Dep();
  var property = Object.getOwnPropertyDescriptor(obj, key);
  if (property && property.configurable === false) {
    return
  }

  var getter = property && property.get;
  var setter = property && property.set;

  var childOb = observe(val);
  Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      var value = getter ? getter.call(obj) : val;
      if (Dep.target) {
        dep.depend();
        if (childOb) {
          childOb.dep.depend();
        }
        if (Array.isArray(value)) {
          dependArray(value);
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      if (newVal === value || (newVal !== newVal && value !== value)) {
        return
      }
      if ("development" !== 'production' && customSetter) {
        customSetter();
      }
      if (setter) {
        setter.call(obj, newVal);
      } else {
        val = newVal;
      }
      childOb = observe(newVal);
      dep.notify();
    }
  });
}
```

之前你有可能听说过vue监听数据变化是利用了```Object.defineProperty```这个API，现在我们终于要讨论作者是如何使用这个API的了。

我们要观察的是obj的key属性，它的初始值是val。作者做的是把这个属性重新定义为getter/setter的形式，这样我们就可以监听到了对这个属性的取值和赋值了。

对象的每一个属性相对应的有一个游离的dep实例，当这个属性变化时(作者竟然考虑了```NaN```这个情况)通过这个dep实例通知到相关的watcher实例(赋值时所做的事情)。


之前我一直在说dep是把数据的变化通知给watcher，那这两者是如何建立联系的呢？这里揭示了一部分：watcher把自身挂载到```Dep.target```上，然后获取依赖的数据，就走到了```get```方法这里。我在上一节提过：如果一个对象(父对象)的属性是个对象(子对象)，那子对象相关联的dep对象有两个，一个依附于observer实例，一个是游离的。当watcher获取依赖时，通过```dep.depend();```游离的dep实例可以和watcher建立联系，通过```childOb.dep.depend();```结合的dep实例建立与watcher的联系。