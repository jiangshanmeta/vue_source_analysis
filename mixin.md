在讲初始化的预处理策略时，有个parent我一直是在淡化处理，现在我们终于要直面它了。

在Vue这个类上有个静态属性```options```，这是parent的第一个来源。作者为```options```预制了几个基本的属性：```filters```、```directives```、```components```等。同时我们可以通过静态方法```Vue.mixin```对这个```options```进行扩展，但这样扩展是全局性的，如果不是特殊需要不建议这么干。

parent的第二个来源是获取vue实例时的mixins字段。


```javascript
if (child.mixins) {
	for (var i = 0, l = child.mixins.length; i < l; i++) {
		parent = mergeOptions(parent, child.mixins[i], vm);
	}
}
```

通过上面的代码，将mixins字段merge到了parent对象上。

这一节的标题是mixin，是时候解释一下这个概念了。在编写应用的时候，有些逻辑是可以复用的，一个方案是组件，但组件相对来说不太灵活，而且粒度可能太小，于是便有了mixin。它将一些小的方法、组件组合成了一个能完成特定功能的整体，我们基于这样的整体搭建功能。