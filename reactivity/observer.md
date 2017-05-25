observer负责观察一个对象/数组，并将对象/数组的变化通过相关联的dep实例通知给watcher实例。

```javascript
var Observer = function Observer (value) {
  this.value = value;
  this.dep = new Dep();
  this.vmCount = 0;
  def(value, '__ob__', this);
  if (Array.isArray(value)) {
    var augment = hasProto
      ? protoAugment
      : copyAugment;
    augment(value, arrayMethods, arrayKeys);
    this.observeArray(value);
  } else {
    this.walk(value);
  }
};
```

这个是Observer的构造函数，需要观察的值以```value```为名挂载在了observer实例上，同时我们可以通过```__ob__```这个属性从被观察的对象找到这个观察者。这样被观察者value和观察者observer就关联在一起了。

observer实例上有一个关联的dep对象(我称之为 结合的dep实例)，我们之后会大量看到observer实例通过dep实例通知watcher实例。之后讲如何观察对象时你可能会发现，一个对象，把它看成一个另一个对象的属性时，他也会有一个关联的dep实例(游离的dep实例，不关联到observer上)。换句话说一个对象会对应两个dep实例。我会在讲```$set```那一节解释为什么要出现结合的dep实例。