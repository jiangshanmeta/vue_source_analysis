vue.js中有一些小函数可以单独摘出来加入到自己的代码库中

#### remove

```javascript
function remove (arr, item) {
  if (arr.length) {
    var index = arr.indexOf(item);
    if (index > -1) {
      return arr.splice(index, 1)
    }
  }
}
```

经常会遇到这么一个需求，在一个数组中删除一个特定的元素，第一步要做的是根据元素找到对应的索引，第二部根据索引删除元素。其实我个人觉得这个封装到Array.prototype上也不为过，毕竟太常用到了。

```javascript
if(!Array.prototype.remove){
	Array.prototype.remove = function(item){
		if(arguments.length===0){
			throw new TypeError('item to be removed is needed');
		}
		var indexOf = Array.prototype.indexOf;
		var splice = Array.prototype.splice;
		var list = Object(this);
		var index = indexOf.call(list,item);
		if(index > -1){
			splice.call(list,index,1);
			return true;
		}
		return false;
	}
}
```

之前讲javascript的polyfill时提到过，Array原型上的```findIndex```方法是个高配版的```indexOf```，它通过传入的回调判断是否满足条件，我们也可以按照这个思路下来加强```remove```方法。

```javascript
if(!Array.prototype.removeBy){
	Array.prototype.removeBy = function(callback,thisArg){
		if (typeof callback !== 'function') {
			throw new TypeError('callback must be a function');
		}
		var findIndex = Array.prototype.findIndex;
		var splice = Array.prototype.splice;
		var list = Object(this);
		var index = findIndex.call(list,callback,thisArg);
		if(index>0){
			splice.call(list,index,1);
			return true;
		}
		return false;
	}
}
```

#### cached

```javascript
function cached (fn) {
  var cache = Object.create(null);
  return (function cachedFn (str) {
    var hit = cache[str];
    return hit || (cache[str] = fn(str))
  })
}
```

这个函数的作用是对传入的函数进行一次包装，使其具有缓存结果的能力。在underscore.js里也有类似的代码：

```javascript
_.memoize = function(func, hasher) {
	var memoize = function(key) {
		var cache = memoize.cache;
		var address = '' + (hasher ? hasher.apply(this, arguments) : key);
		if (!_.has(cache, address)) cache[address] = func.apply(this, arguments);
			return cache[address];
	};
	memoize.cache = {};
	return memoize;
};
```

基本思路是一致的，都是找个变量缓存结果，但是underscore还支持了定制缓存时的键。


#### camelize

```javascript
var camelizeRE = /-(\w)/g;
var camelize = cached(function (str) {
	return str.replace(camelizeRE, function (_, c) { return c ? c.toUpperCase() : ''; })
});
``` 

这个函数的作用是把中划线连接的字符串转换成驼峰的形式，vue还对驼峰化的结果进行了缓存。我们可以把这里的缓存功能去掉作为一个工具函数。

```javascript
var camelize = (function(){
	var camelizeRE = /-(\w)/g;
	return function(str){
		return str.replace(camelizeRE,function (_, c) { return c ? c.toUpperCase() : ''; })
	}
})();
```

#### hyphenate

这个函数所做的和上面的```camelize```正好相反，它是把驼峰化的字符串转化为中划线连接的。

我们先看下vue中的实现：

```javascript
var hyphenateRE = /([^-])([A-Z])/g;
var hyphenate = cached(function (str) {
	return str
		.replace(hyphenateRE, '$1-$2')
		.replace(hyphenateRE, '$1-$2')
		.toLowerCase()
});
```

我们结合实例看一下作者的思路，比如字符串```aBcD```，我们期望的中划线形式是```a-bc-d```，经过第一个replace方法我们得到了```a-Bc-D```，到这里我们只需转一下大小写就好了，同时注意到在这个情况下第二个replace是没有任何匹配结果的。那第二个replace是做什么，或者说什么情况下需要第二个replace，我的答案是字符串有连续大写字母的情况。比如字符串```aBCD```，第一个replace的结果是```a-BC-D```(请思考为什么)，经过第二个replace我们得到```a-B-C-D```。

关于中划线化这个方法，我想提出我的看法：

```javascript
function hyphenate(str){
	var hyphenateRE = /([A-Z])/g;
	return str.replace(hyphenateRE,'-$1').toLowerCase();
}
```

我是这样考虑的：中划线化需要每个大写字母前面加个中划线，然后转小写即可。我能想到的和vue的方案有区别的情况是首字母是大写。vue的方案首字母大写不会在首字母前面加中划线，我的方案会加，但驼峰化的字符串一般而言首字母都是小写的啊。


#### capitalize

```javascript
var capitalize = cached(function (str) {
	return str.charAt(0).toUpperCase() + str.slice(1)
});
```

在php里有个同样功能的函数叫```ucfirst```，就是把字符串首字母大写。



#### isReserved

```javascript
function isReserved (str) {
	var c = (str + '').charCodeAt(0);
	return c === 0x24 || c === 0x5F
}
```

一般有个默认的规范是以```_```开头的为私有，在vue中原型上的公有方法以```$```开头，因而这两个作为保留字


#### isNative

```javascript
function isNative (Ctor) {
	return typeof Ctor === 'function' && /native code/.test(Ctor.toString())
}
```

这里```native code```的含义是：native代码实现的build-in函数，而不是javascript代码。毕竟有时希望原生的功能，而不是其他开发者polyfill的功能。