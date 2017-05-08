在Vue.js中，除了对外暴露的Vue类之外，还有Dep类、Observer类、Watcher类这几个辅助类。

Dep类是其中最轻的一个类，它所起的作用是连接起data和watcher实例，当数据更新时通知相关的watcher实例。


```javascript
var uid = 0;
var Dep = function Dep () {
	this.id = uid++;
	this.subs = [];
};

Dep.prototype.addSub = function addSub (sub) {
	this.subs.push(sub);
};

Dep.prototype.removeSub = function removeSub (sub) {
	remove(this.subs, sub);
};

Dep.prototype.depend = function depend () {
	if (Dep.target) {
		Dep.target.addDep(this);
	}
};

Dep.prototype.notify = function notify () {
  	// stabilize the subscriber list first
	var subs = this.subs.slice();
	for (var i = 0, l = subs.length; i < l; i++) {
		subs[i].update();
	}
};
```

上面差不多把Dep对象上的全部方法列出来了，最核心的是维护了一个订阅者序列```subs```，```addSub```方法和```depend```用来添加相关watcher实例(这两个方法的区别我会在watcher类中说明，同时也会说明为dep实例添加唯一id的原因)，```removeSub```用来移除订阅者，其中的```remove```方法我在[Vue的基础工具函数](https://jiangshanmeta.gitbooks.io/vue_source_analysis/content/common.html)已经提及了。最核心的一个方法是```notify```，它将数据的变化通知到了watcher实例。你可能会好奇在```notify```的实现中作者为什么要先固化订阅者。其实一开始我也没想明白，后来看了[```$emit```方法的实现](https://jiangshanmeta.gitbooks.io/vue_source_analysis/content/event/emit.html)才想明白，因为watcher的更新可能会改变这个订阅序列。

还有一个问题是dep实例和watcher实例之间的关系是如何建立的。他们之间是通过```Dep.target```关联在一起的，具体的要放到Watcher类去讲。