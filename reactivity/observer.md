Observer类负责将数据转换为响应式的。Observer所处理的数据是对象/数组，所以它需要一个入口提前处理异常值。

```javascript
// 获得observer实例的入口
function observe (value, asRootData) {
  // 筛出异常值
  if (!isObject(value)) {
    return
  }
  var ob;
  // 数据已经建立关联的observer实例就不再重复建立
  if (hasOwn(value, '__ob__') && value.__ob__ instanceof Observer) {
    ob = value.__ob__;
  } else if (
    observerState.shouldConvert &&
    !isServerRendering() &&
    (Array.isArray(value) || isPlainObject(value)) &&
    Object.isExtensible(value) &&
    !value._isVue
  ) {
    ob = new Observer(value);
  }
  if (asRootData && ob) {
    ob.vmCount++;
  }
  return ob
}
```

Observer与数据相互关联，然后将数据转化为响应式的：

```javascript
var Observer = function Observer (value) {
  // observer关联到数据
  this.value = value;
  this.dep = new Dep();
  // vmCount一个用途是表明是否是根数据元素，TODO
  this.vmCount = 0;
  // 数据关联到observer
  def(value, '__ob__', this);
  // 数组和对象分别处理
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
// 对object的属性观察
Observer.prototype.walk = function walk (obj) {
  var keys = Object.keys(obj);
  for (var i = 0; i < keys.length; i++) {
    defineReactive$$1(obj, keys[i], obj[keys[i]]);
  }
};
// 对array的元素观察
Observer.prototype.observeArray = function observeArray (items) {
  for (var i = 0, l = items.length; i < l; i++) {
    observe(items[i]);
  }
};
```

我们先看如何把对象的元素变成响应式的：

```javascript
function defineReactive$$1 (obj,key,val,customSetter) {
  // 与这个元素关联的dep实例
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
      // 收集依赖的原理
      if (Dep.target) {
        dep.depend();
        // childOb对应的dep也要收集依赖，因为两个dep反应的是同一个对象与watcher的关系
        if (childOb) {
          childOb.dep.depend();
        }
        // 值为数组，深度观察
        if (Array.isArray(value)) {
          dependArray(value);
        }
      }
      return value
    },
    set: function reactiveSetter (newVal) {
      var value = getter ? getter.call(obj) : val;
      // 数据变化了才更新，考虑了NaN的情况
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
      // 重新观察数据
      childOb = observe(newVal);
      // 通过dep通知watcher数据更新
      dep.notify();
    }
  });
}
```

可能你已经听说过Vue使用```Object.defineProperty```检测数据变化，上面就是对应的核心实现。在get中dep与watcher建立关联，在set中dep通知watcher数据更新。


这种检测数据变化的方式有个问题，它无法观察数组的变化，Vue采取的解决方案是重载```push```、```pop```等常用数组方法。

```javascript
var arrayProto = Array.prototype;
var arrayMethods = Object.create(arrayProto);
[
  'push',
  'pop',
  'shift',
  'unshift',
  'splice',
  'sort',
  'reverse'
]
.forEach(function (method) {
  // 缓存原始方法
  var original = arrayProto[method];
  // 定义重载的新方法
  def(arrayMethods, method, function mutator () {
    var arguments$1 = arguments;

    // avoid leaking arguments:
    // http://jsperf.com/closure-with-arguments
    var i = arguments.length;
    var args = new Array(i);
    while (i--) {
      args[i] = arguments$1[i];
    }
    // 调用原始的方法
    var result = original.apply(this, args);
    var ob = this.__ob__;
    var inserted;
    switch (method) {
      case 'push':
        inserted = args;
        break
      case 'unshift':
        inserted = args;
        break
      case 'splice':
        inserted = args.slice(2);
        break
    }
    // 观察新添加的数据
    if (inserted) { ob.observeArray(inserted); }

    // observer实例上dep实例通知watcher数据更新
    ob.dep.notify();
    return result
  });
});
var hasProto = '__proto__' in {};
// 调用见Observer的构造函数
// 相当于Object.setPrototypeOf方法
function protoAugment (target, src) {
  target.__proto__ = src;
}

// 调用见Observer的构造函数
function copyAugment (target, src, keys) {
  for (var i = 0, l = keys.length; i < l; i++) {
    var key = keys[i];
    def(target, key, src[key]);
  }
}
```

要理解上面这段代码，需要理解原型链的原理，在此基础上才能明白```arrayMethods```这个对象是如何起作用的。```protoAugment```方法是在要观察的数组和```Array.prototype```之间插了一层，当访问```push```等方法时，会使用```arrayMethods```上定义的相关方法。```copyAugment```是把```arrayMethods```上重载的几个方法关联到了要观察的数组上，当访问```push```等方法时，会直接访问自身的```push```方法。


对于一个对象，它可以有两个相关的dep实例，一个在defineReactive函数中生成(以下称为游离的dep实例)(更准确的说法是和键相关联的dep)，一个是在observer构造函数中生成(以下称为结合的dep实例)，因为这两个dep反应的数据是一个，所以对应的watcher也应一致，所以才有了```if (childOb) {childOb.dep.depend();}```。当为这个对象增加($set)或删除属性($delete)时，会通过结合的dep实例通知watcher。


对于一个数组，它只有一个相关的dep实例，是挂在observer上的，数组数据更新(在重载的变异方法中)需要通过这个dep实例通知watcher数据变化。