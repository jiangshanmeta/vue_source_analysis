patch的意思是打补丁，包含一种在原有基础上修改的意思。但是Vue的patch包含了三层意思：创建、修改、删除。目前先走马观花看一下patch过程。

这里有个基本概念**sameVnode**


```javascript
function sameVnode (a, b) {
	return (
		a.key === b.key &&
		a.tag === b.tag &&
		a.isComment === b.isComment &&
		isDef(a.data) === isDef(b.data) &&
		sameInputType(a, b)
	)
}
function sameInputType (a, b) {
	if (a.tag !== 'input') { return true }
	var i;
	var typeA = isDef(i = a.data) && isDef(i = i.attrs) && i.type;
	var typeB = isDef(i = b.data) && isDef(i = i.attrs) && i.type;
	return typeA === typeB
}
```

虽然名字是sameVnode，但是判断标准相对较松，翻译成*值得比较节点*比较合适。它要求两个节点的tag一致、key一致、如果是input还要type一致，如果满足这些条件，就认为可以考虑在原有基础上修改。


## 创建

当旧vnode是真实DOM时，这意味着我们是处于生命周期的mount阶段。我们需要将新vnode映射为真实DOM，并且替换旧DOM。

## 更新

这对应生命周期的update阶段。当新旧vnode不值得比较时，会直接创建新的DOM替换掉旧DOM。



当新旧两个节点值得比较时，会调用patchVnode更新DOM，这是patch的重点。

```javascript
function patchVnode (oldVnode, vnode, insertedVnodeQueue, removeOnly) {
	if (oldVnode === vnode) {
		return
	}

	// once这样能重复利用就重复利用
	if (isTrue(vnode.isStatic) &&
	    isTrue(oldVnode.isStatic) &&
	    vnode.key === oldVnode.key &&
	    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))) {
			vnode.elm = oldVnode.elm;
			vnode.componentInstance = oldVnode.componentInstance;
			return
	}
	var i;
	var data = vnode.data;
	if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
	  i(oldVnode, vnode);
	}
	var elm = vnode.elm = oldVnode.elm;
	var oldCh = oldVnode.children;
	var ch = vnode.children;

	// update vnode相关属性
	if (isDef(data) && isPatchable(vnode)) {
	  for (i = 0; i < cbs.update.length; ++i) { cbs.update[i](oldVnode, vnode); }
	  if (isDef(i = data.hook) && isDef(i = i.update)) { i(oldVnode, vnode); }
	}

	// 处理子节点
	if (isUndef(vnode.text)) {
	  // 新vnode和老vnode都有子节点，updateChildren处理	
	  if (isDef(oldCh) && isDef(ch)) {
	    if (oldCh !== ch) { updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly); }
	  } else if (isDef(ch)) {
	  	// 只有新vnode有子节点，添加对应节点即可
	    if (isDef(oldVnode.text)) { nodeOps.setTextContent(elm, ''); }
	    addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue);
	  } else if (isDef(oldCh)) {
	  	// 只有旧vnode有子节点，干掉这些子节点
	    removeVnodes(elm, oldCh, 0, oldCh.length - 1);
	  } else if (isDef(oldVnode.text)) {
	    nodeOps.setTextContent(elm, '');
	  }
	} else if (oldVnode.text !== vnode.text) {
		// 更新文本节点
	  nodeOps.setTextContent(elm, vnode.text);
	}
	if (isDef(data)) {
	  if (isDef(i = data.hook) && isDef(i = i.postpatch)) { i(oldVnode, vnode); }
	}
}

```

patchVnode是从vnode更新视图的核心方法。首先处理了once这样的特殊情况，保证复用DOM元素，然后更新data属性，接着处理子节点。如果新vnode有子节点而旧vnode没有子节点，直接添加对应节点；如果没有新子节点只有旧子节点，干掉旧子节点即可；最为复杂的情况是新旧vnode均有子节点，那么需要调用updateChildren方法，关于这个方法，[请看aooy的博客](https://github.com/aooy/blog/issues/2)。


## 销毁

当没有新的vnode只有旧的vnode时，意味着我们要销毁旧vnode，这时候调用invokeDestroyHook销毁旧节点，这对应生命周期的destroy阶段。