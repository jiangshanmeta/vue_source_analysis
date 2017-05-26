Dep类是用来将数据和watcher关联起来的一个类。


```javascript
var uid = 0;

var Dep = function Dep () {
  this.id = uid++;
  this.subs = [];
};

// 添加对某个watcher的关联
Dep.prototype.addSub = function addSub (sub) {
  this.subs.push(sub);
};

// 解除对某个watcher的关联
Dep.prototype.removeSub = function removeSub (sub) {
  remove(this.subs, sub);
};

// dep和watcher相互关联的入口
Dep.prototype.depend = function depend () {
  // Dep.target 期望存放watcher实例
  if (Dep.target) {
  	// watcher添加对dep的关联，同时在内部调用dep的addSub方法，添加dep对watcher的关联
    Dep.target.addDep(this);
  }
};

// 通知watcher数据更新
Dep.prototype.notify = function notify () {
  var subs = this.subs.slice();
  for (var i = 0, l = subs.length; i < l; i++) {
    subs[i].update();
  }
};
```

每一个dep实例都会有一个唯一的id，这个唯一标示会在watcher实例中用到。dep实例的核心是```subs```字段，它维护一个watcher队列，当时据变化时通知watcher数据更新(notify方法)。

deps和watcher是相互关联的，可以想到watcher也会维护一个deps队列。dep和watcher建立联系的入口是```depend```方法，你可能会好奇为什么要有这么个入口，你可能试着去掉```depend```方法然后把相互关联功能放到addSub方法中：

```javascript
Dep.prototype.addSub = function(){
  var watcher = Dep.target;
	if(!watcher){
		return;
	}
	// 判断watcher是否关联到了dep
	if(watcher.deps.indexOf(this)===-1){
		watcher.deps.push(this);
	}
	// 判断dep是否关联到了watcher
	if(this.subs.indexOf(watcher)===-1){
		this.subs.push(watcher);
	}
}
```

这么做确实可行，但是这么做时间上比较吃亏(判断是否建立了关系需要O(n))，所以作者把是否建立了联系这个判断放在了watcher中，利用集合实现(时间O(1))。

```javascript
Watcher.prototype.addDep = function addDep (dep) {
  // newDepIds和depIds都是集合
  var id = dep.id;
  // 判断watcher是否关联到了dep
  if (!this.newDepIds.has(id)) {
    this.newDepIds.add(id);
    this.newDeps.push(dep);
    // 判断dep是否关联到了watcher
    if (!this.depIds.has(id)) {
      dep.addSub(this);
    }
  }
};
```