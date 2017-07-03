虚拟节点是用来模拟真实节点的，下面是Vue中虚拟节点的构造函数：

```javascript
var VNode = function VNode (
  tag,
  data,
  children,
  text,
  elm,
  context,
  componentOptions
) {
  this.tag = tag;
  this.data = data;
  this.children = children;
  this.text = text;
  this.elm = elm;
  this.ns = undefined;
  this.context = context;
  this.functionalContext = undefined;
  this.key = data && data.key;
  this.componentOptions = componentOptions;
  this.componentInstance = undefined;
  this.parent = undefined;
  this.raw = false;
  this.isStatic = false;
  this.isRootInsert = true;
  this.isComment = false;
  this.isCloned = false;
  this.isOnce = false;
};
```

* tag 表明节点的标签类型
* data 包含相关attr、事件等
* children 存放的是子虚拟节点
* text 是文本节点专用的，表明文本内容
* ele 虚拟节点对应的真实DOM节点
* ns html的命名空间(用处不大)
* context
* functionalContext
* key 对一个节点的唯一标识
* componentOptions
* componentInstance
* parent
* raw
* isStatic 是否是静态的(更新时可以跳过diff步骤)
* isRootInsert
* isComment 是否是comment节点
* isCloned 是否是克隆的节点
* isOnce 是否是一次性节点(更新时可以跳过diff步骤)

基于这个构造函数，作者封装了一些获取特殊类型vnode实例的方法：

```javavscript
// 创建一个空的vnode实例
var createEmptyVNode = function () {
	var node = new VNode();
	node.text = '';
	node.isComment = true;
	return node
};

// 创建一个文本vnode
function createTextVNode (val) {
  return new VNode(undefined, undefined, undefined, String(val))
}

// 克隆一个vnode
function cloneVNode (vnode) {
	var cloned = new VNode(
		vnode.tag,
		vnode.data,
		vnode.children,
		vnode.text,
		vnode.elm,
		vnode.context,
		vnode.componentOptions
	);
	cloned.ns = vnode.ns;
	cloned.isStatic = vnode.isStatic;
	cloned.key = vnode.key;
	cloned.isCloned = true;
	return cloned
}

// 克隆多个vnode
function cloneVNodes (vnodes) {
	var len = vnodes.length;
	var res = new Array(len);
	for (var i = 0; i < len; i++) {
		res[i] = cloneVNode(vnodes[i]);
	}
	return res
}

// 根据一个DOM节点创建vnode
function emptyNodeAt (elm) {
	return new VNode(nodeOps.tagName(elm).toLowerCase(), {}, [], undefined, elm)
}
```

