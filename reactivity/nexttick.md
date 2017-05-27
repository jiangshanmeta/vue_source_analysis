在讲Watcher类的时候，提到有一个```sync```字段，这个字段表示dep通知watcher更新的时候，watcher是同步更新还是异步更新。关于异步更新，Vue暴露了两个API：```Vue.nextTick```和```Vue.prototype.$nextTick```，这两个API都依赖于下面一段代码：

```javascript
var nextTick = (function () {
  // 回调队列
  var callbacks = [];

  // 是否在等待执行任务
  var pending = false;

  // 异步实现函数，异步调用nextTickHandler
  var timerFunc;

  // 执行回调队列中的回调
  function nextTickHandler () {
    // 允许调用timerFunc
    pending = false;
    var copies = callbacks.slice(0);
    // 清空回调队列
    callbacks.length = 0;
    // 执行回调，执行中可能添加新回调，因而长度不能缓存
    for (var i = 0; i < copies.length; i++) {
      copies[i]();
    }
  }

  // 针对不同浏览器使用不同异步实现方案
  if (typeof Promise !== 'undefined' && isNative(Promise)) {
    var p = Promise.resolve();
    var logError = function (err) { console.error(err); };
    timerFunc = function () {
      p.then(nextTickHandler).catch(logError);
      if (isIOS) { setTimeout(noop); }
    };
  } else if (typeof MutationObserver !== 'undefined' && (
    isNative(MutationObserver) ||
    MutationObserver.toString() === '[object MutationObserverConstructor]'
  )) {
    var counter = 1;
    var observer = new MutationObserver(nextTickHandler);
    var textNode = document.createTextNode(String(counter));
    observer.observe(textNode, {
      characterData: true
    });
    timerFunc = function () {
      counter = (counter + 1) % 2;
      textNode.data = String(counter);
    };
  } else {
    timerFunc = function () {
      setTimeout(nextTickHandler, 0);
    };
  }

  return function queueNextTick (cb, ctx) {
    var _resolve;
    // 向回调队列添加任务
    callbacks.push(function () {
      if (cb) {
        try {
          cb.call(ctx);
        } catch (e) {
          handleError(e, ctx, 'nextTick');
        }
      } else if (_resolve) {
        _resolve(ctx);
      }
    });
    // timerFunc一次调用就会异步处理完整个回调队列
    // 所以在此期间只需要向任务队列添回调，无需重复调用timerFunc
    if (!pending) {
      pending = true;
      timerFunc();
    }
    if (!cb && typeof Promise !== 'undefined') {
      return new Promise(function (resolve, reject) {
        _resolve = resolve;
      })
    }
  }
})();
```

为了实现异步，Vue会依次尝试```Promise```方案、```MutationObserver```方案和```setTimeout```方案，并最终暴露出一个```timerFunc```。调用```timerFunc```后，会异步执行完回调队列(callbacks)内的回调。


对于watcher来说，一般更新是异步的，这样可以实现即使一个watcher被多次触发，也仅会向队列中添加一次，避免不必要的运算。在watcher的```update```方法中，会调用```queueWatcher```将watcher推入队列。


```javascript
// 用于快速判断watcher是否已经加入了queue
var has = {};

// 表示是否正在处理queue中的watcher
var flushing = false;

// 正在处理的watcher在queue的索引
var index = 0;

// 类似于nextTick的pending
var waiting = false;

// 待更新的watcher队列
var queue = [];

function queueWatcher (watcher) {
  var id = watcher.id;
  // 去重，避免多次更新
  if (has[id] == null) {
    has[id] = true;

    // 没有处理watcher队列，向队列插入
    if (!flushing) {
      queue.push(watcher);
    } else {
      // 如果正在处理watcher队列，将watcher加入到合适的位置
      var i = queue.length - 1;
      while (i >= 0 && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(Math.max(i, index) + 1, 0, watcher);
    }
    // flushSchedulerQueue执行一次会处理整个queue
    if (!waiting) {
      waiting = true;
      nextTick(flushSchedulerQueue);
    }
  }
}
```

queueWatcher主要完成的是将watcher推入一个队列，但是这个推入的位置比较有讲究。如果watcher队列没有正在处理，push进去即可(反正最后还要重排)，如果watcher队列被正在处理，显然插入的位置要在当前处理位置(index)之后，由于此时watcher队列中的watcher已经按照升序排好了，新加入watcher要保证未处理的watcher依然按照id增序排列，所以才有了```queue.splice(Math.max(i, index) + 1, 0, watcher);```，这也意味着即使watcher队列在处理中，其长度也是可以改变的。


```javascript
// 处理watcher队列
function flushSchedulerQueue () {
  // 开始处理watcher队列
  flushing = true;
  var watcher, id;

  // 按照id升序对watcher排序
  queue.sort(function (a, b) { return a.id - b.id; });

  // 遍历处理watcher，不缓存队列的长度，因为处理过程中可能有新的watcher加入
  for (index = 0; index < queue.length; index++) {
    watcher = queue[index];
    id = watcher.id;
    has[id] = null;

    // 更新watcher的值，处理watcher的回调，最核心的一句
    watcher.run();

    if ("development" !== 'production' && has[id] != null) {
      circular[id] = (circular[id] || 0) + 1;
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? ("in watcher with expression \"" + (watcher.expression) + "\"")
              : "in a component render function."
          ),
          watcher.vm
        );
        break
      }
    }
  }

  
  var activatedQueue = activatedChildren.slice();
  var updatedQueue = queue.slice();

  // 相关属性重置
  resetSchedulerState();


  callActivatedHooks(activatedQueue);

  // 对于负责render的watcher，要调用update钩子
  callUpdateHooks(updatedQueue);


  if (devtools && config.devtools) {
    devtools.emit('flush');
  }
}

// 重置watcher 队列和相关属性
function resetSchedulerState () {

  queue.length = activatedChildren.length = 0;
  has = {};
  {
    circular = {};
  }
  waiting = flushing = false;
}
```

处理watcher队列时有一步是对watcher队列按照id升序排序，作者提到了以下三个原因：

* 保证先父元素再子元素
* 保证用户创建的watcher(computed和watch的)先于负责render的watcher
* 如果父组件watcher更新时，子组件watcher被销毁，可以跳过这个子组件的watcher