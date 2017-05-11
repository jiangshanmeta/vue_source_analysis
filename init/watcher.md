Watcher是一个用的比较广泛的类，就目前我所知，它用在了computed和watch的实现上。我们先看一下它的构造函数：

```javascript
var Watcher = function Watcher (vm,expOrFn,cb,options) {
  // 与vue实例相互关联
  this.vm = vm;
  vm._watchers.push(this);
  // options
  if (options) {
    this.deep = !!options.deep;
    this.user = !!options.user;
    this.lazy = !!options.lazy;
    this.sync = !!options.sync;
  } else {
    this.deep = this.user = this.lazy = this.sync = false;
  }
  this.cb = cb;
  this.id = ++uid$2; // uid for batching
  this.active = true;
  this.dirty = this.lazy; // for lazy watchers

  // 存放相关dep实例
  this.deps = [];
  this.newDeps = [];

  // 存放相关dep实例的id，用集合实现
  this.depIds = new _Set();
  this.newDepIds = new _Set();
  this.expression = expOrFn.toString();
  // parse expression for getter

  // 数据获取函数
  if (typeof expOrFn === 'function') {
    this.getter = expOrFn;
  } else {
    this.getter = parsePath(expOrFn);
    if (!this.getter) {
      this.getter = function () {};
      "development" !== 'production' && warn(
        "Failed watching path: \"" + expOrFn + "\" " +
        'Watcher only accepts simple dot-delimited paths. ' +
        'For full control, use a function instead.',
        vm
      );
    }
  }
  this.value = this.lazy
    ? undefined
    : this.get();
};
```

知道了watcher实例上有哪些属性之后，我们需要回答以下几个问题：

* watcher和dep是如何建立联系的
* watcher是如何更新与dep建立的联系

#### watcher和dep是如何建立联系的

```javascript
Watcher.prototype.get = function get () {
  pushTarget(this);
  var value;
  var vm = this.vm;
  if (this.user) {
    try {
      value = this.getter.call(vm, vm);
    } catch (e) {
      handleError(e, vm, ("getter for watcher \"" + (this.expression) + "\""));
    }
  } else {
    value = this.getter.call(vm, vm);
  }
  // "touch" every property so they are all tracked as
  // dependencies for deep watching
  if (this.deep) {
    traverse(value);
  }
  popTarget();
  this.cleanupDeps();
  return value
};

Dep.target = null;
var targetStack = [];

function pushTarget (_target) {
  if (Dep.target) { targetStack.push(Dep.target); }
  Dep.target = _target;
}

function popTarget () {
  Dep.target = targetStack.pop();
}
```

watcher实例先把自身挂到了```Dep.target```上，然后调用```this.getter.call(vm, vm)```去获取相关的数据，在相应数据的getter函数中(在观察对象那一节)，dep和watcher建立联系。

#### watcher是如何更新与dep建立的联系

与这个问题相关的还有为什么有了deps属性，还要有newDeps属性？为什么dep添加依赖有```depend```和```addSub```两个方法？

```javascript
Watcher.prototype.addDep = function addDep (dep) {
  var id = dep.id;
  if (!this.newDepIds.has(id)) {
    // watcher建立与dep的关联
    this.newDepIds.add(id);
    this.newDeps.push(dep);
    // dep建立到watcher的关联
    if (!this.depIds.has(id)) {
      dep.addSub(this);
    }
  }
};
Watcher.prototype.cleanupDeps = function cleanupDeps () {
  var this$1 = this;

  var i = this.deps.length;
  // watcher现在不需要依赖的dep，dep要切断和watcher的关系
  while (i--) {
    var dep = this$1.deps[i];
    if (!this$1.newDepIds.has(dep.id)) {
      dep.removeSub(this$1);
    }
  }
  var tmp = this.depIds;
  this.depIds = this.newDepIds;
  this.newDepIds = tmp;
  this.newDepIds.clear();
  tmp = this.deps;
  this.deps = this.newDeps;
  this.newDeps = tmp;
  this.newDeps.length = 0;
};
```

对于watcher而言，当需要更新依赖时，它先把新依赖暂存，然后与旧依赖对比，新依赖的dep实例添加对watcher的关系，watcher不需要依赖的dep，需要dep切断和watcher的关系。```depIds```和```newDepIds```用了集合来存储depId，是为了高效判断是否包含depId（O(1)）。