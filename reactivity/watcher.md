在Vue的各个相关类中，Watcher类起到了中心枢纽的功能，响应式数据的变化通过dep通知到watcher，watcher通过其原型上的各个方法和回调函数实现computed、watch、渲染视图等功能。


```javascript
var uid$2 = 0;

var Watcher = function Watcher (vm,expOrFn,cb,options) {
  // 与vue实例互相关联
  this.vm = vm;
  vm._watchers.push(this);

  if (options) {
    // deep参数为了监听更深层级值的变化
    this.deep = !!options.deep;
    // user参数表明cb是用户传入的，需要注意容错
    this.user = !!options.user;
    // lazy用在computed实现上，dep通知数据更新后仅仅标识当前数据是脏数据
    this.lazy = !!options.lazy;
    // sync参数表示dep通知数据更新后是否同步更新
    this.sync = !!options.sync;
  } else {
    this.deep = this.user = this.lazy = this.sync = false;
  }
  // 数据更新后要执行的回调
  this.cb = cb;
  // 唯一id
  this.id = ++uid$2;
  // 表示这个watcher是否还有用
  this.active = true;

  // 标记lazy watcher初始数据为脏
  this.dirty = this.lazy; 

  // 和dep相关联需要的几个属性
  this.deps = [];
  this.newDeps = [];
  this.depIds = new _Set();
  this.newDepIds = new _Set();

  // 调试时用到
  this.expression = expOrFn.toString();


  // getter是值获取函数
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
  // watcher的值
  this.value = this.lazy? undefined : this.get();
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

// 实现依赖收集，获得计算值
Watcher.prototype.get = function get () {
  // 将该watcher挂在Dep.target上
  pushTarget(this);
  var value;
  var vm = this.vm;
  // 因为是用户传进来的getter，容错
  if (this.user) {
    try {
      value = this.getter.call(vm, vm);
    } catch (e) {
      handleError(e, vm, ("getter for watcher \"" + (this.expression) + "\""));
    }
  } else {
    value = this.getter.call(vm, vm);
  }

  // deep所做的就是递归，让更深的dep关联到watcher
  if (this.deep) {
    traverse(value);
  }
  popTarget();

  // 完成依赖的更新
  this.cleanupDeps();
  return value
};

// 添加依赖
Watcher.prototype.addDep = function addDep (dep) {
  var id = dep.id;
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id);
    this.newDeps.push(dep);
    if (!this.depIds.has(id)) {
      dep.addSub(this);
    }
  }
};

// 解除不再需要的dep和watcher的关系
Watcher.prototype.cleanupDeps = function cleanupDeps () {
    var this$1 = this;

  // 消除不再需要的dep到watcher的关系
  var i = this.deps.length;
  while (i--) {
    var dep = this$1.deps[i];
    if (!this$1.newDepIds.has(dep.id)) {
      dep.removeSub(this$1);
    }
  }

  // 更新当前的依赖
  var tmp = this.depIds;
  this.depIds = this.newDepIds;
  this.newDepIds = tmp;
  this.newDepIds.clear();
  tmp = this.deps;
  this.deps = this.newDeps;
  this.newDeps = tmp;
  this.newDeps.length = 0;
};


// dep通知watcher时的入口函数
Watcher.prototype.update = function update () {
  // lazy watcher 仅仅表示数据是脏的，必要的时候通过evaluate获取数据
  if (this.lazy) {
    this.dirty = true;
  } 
  // 同步更新的watcher 同步调用回调
  else if (this.sync) {
    this.run();
  } 
  // 异步更新的watcher加入到更新队列
  else {
    queueWatcher(this);
  }
};

// 调用回调
Watcher.prototype.run = function run () {
  if (this.active) {
    var value = this.get();
    if (
      value !== this.value || isObject(value) || this.deep) {
      // 设定新值
      var oldValue = this.value;
      this.value = value;
      // 执行回调，还要容错
      if (this.user) {
        try {
          this.cb.call(this.vm, value, oldValue);
        } catch (e) {
          handleError(e, this.vm, ("callback for watcher \"" + (this.expression) + "\""));
        }
      } else {
        this.cb.call(this.vm, value, oldValue);
      }
    }
  }
};

// 用于lazy watcher
Watcher.prototype.evaluate = function evaluate () {
  this.value = this.get();
  this.dirty = false;
};

// watcher 嵌套引用时候需要
Watcher.prototype.depend = function depend () {
    var this$1 = this;

  var i = this.deps.length;
  while (i--) {
    this$1.deps[i].depend();
  }
};


// 解除和vue实例、dep实例的关系，标记为不再活动
Watcher.prototype.teardown = function teardown () {
  var this$1 = this;

  if (this.active) {
    // 解除vm对watcher的关系
    if (!this.vm._isBeingDestroyed) {
      remove(this.vm._watchers, this);
    }
    // 解除dep对watcher的依赖
    var i = this.deps.length;
    while (i--) {
      this$1.deps[i].removeSub(this$1);
    }
    this.active = false;
  }
};
```

Watcher原型链上的方法看起来比较多，我们可以先分下类：```get```、```addDep```、```cleanupDeps```这三个方法是用来与dep互相关联使用的，```get```和```evaluate```是用来获取watcher的值的,```update```是暴露出去的让dep通知的入口，```run```方法是执行watcher的回调的入口，最后一个```teardown```用来取消观察停止回调。

到这里我们已经掌握了依赖收集机制必要的知识，现在我们理一遍流程：当watcher要收集依赖时，它会调用```get```方法，把watcher本身挂载到```Dep.target```上，然后watcher调用```getter```函数，我们去获取响应式的属性，这样就走到了属性的get方法(defineReactive方法的结果)，在属性的get方法中，dep调用```depend```方法，```depend```方法再调用watcher的```addDep```方法，这样watcher和dep就建立了联系，最后watcher调用```cleanupDeps```方法，把那些曾经关联到watchr但现在不再关联到watcher的dep的到watcher的关系取消掉。


之前一直没想明白为什么要用```targetStack```。考虑这么一个场景：computed属性A需要用到computed属性B，调用A对应watcher(watcherA)的get方法时，会先把A对应watcher先挂到```Dep.target```上，然后会调用watcherA的getter方法，在getterA的某个阶段会访问属性B，B的watcher也会调用对应的get方法，也需要收集依赖，这时我们需要暂存watcherA(入targetStack)，然后把watcherB挂到```Dep.target```上。这种嵌套依赖需要```targetStack```辅助。还有一个问题，watcherB依赖的Dep实例，watcherA是否也需要依赖？答案是肯定的，毕竟watcherB不能把自身变化通知给watcherA，负责这个的是watcher原型链上的```depend```方法。在computed的具体实现中我们会看到它是如何运用的。