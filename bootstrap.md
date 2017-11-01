最开始读Vue源码的时候是读的编译后的代码，因为那时没接触过前端工程化的东西，使用Vue是通过```<script>```标签直接引入，而不是包管理工具。虽然如此依然是读完了Vue的响应式原理。然而这么读真的很费劲，所以我就准备读真·源码了。读源码其实最忌讳一开始就深入细节而不关注整体结构(underscore这种各个方法几乎独立的小库除外)，所以这里要介绍的就是项目整体结构。

我读的是2.3.0版本，一些后续版本添加的feature我后面也会补充上。

我们真正需要关心的是src目录下的代码，它包括以下几个子目录：

* compiler 包含将template转换为render函数的代码
* core 和平台无关的核心代码，比如vue的响应式、虚拟DOM
* platforms 和平台有关的代码，入口文件也在这个目录下
* server 服务器端渲染相关
* sfc 单文件组件相关
* shared 共享的代码，比如一些工具方法

那我们的入口文件在哪？上面提到了在platforms下面，我们暂时略过weex直接看web目录。这个目录下有多个js文件，哪个是入口文件啊？其实都是。vue的代码大概分为两块，一块是compiler，大概负责将模板转换为render函数，另一块是runtime，用来创建Vue实例、渲染以及处理虚拟DOM。这样就对应了三个入口:compier.js(只含有compiler代码)，runtime.js(只含有runtime代码)以及runtime-with-compiler.js(两块代码都包含)。还有一个server-render.js是负责服务器端渲染的。这三个入口，结合不同的构建format，就构成了[文档中提及的不同构建版本](https://cn.vuejs.org/v2/guide/installation.html#对不同构建版本的解释)。更为具体的描述需要看build相关代码(尤其是build目录下的config.js)，然而直接把runtime-with-compiler.js当成入口文件就已经足够了。

其他和src平级的目录，build目录负责构建，dist目录是构建的目标目录，flow是类型声明相关代码(使用flow做类型限制)，types目录是用typescript声明类型，这些没有必要做过多深究，知道干什么的就好了。