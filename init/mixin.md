mixin本质上是props、data、method、computed、watch、component、directive、filter的大杂烩，在地位上低于component，一般是作为组件的组成部分来使用。

对于实例选项中mixin的预处理，实质上是对props等选项的预处理，

```javascript
// 运用上一节提到的预处理策略合并选项
export function mergeOptions (
  parent: Object,
  child: Object,
  vm?: Component
): Object {
  if (process.env.NODE_ENV !== 'production') {
    checkComponents(child)
  }

  if (typeof child === 'function') {
    child = child.options
  }

  normalizeProps(child)
  normalizeDirectives(child)
  const extendsFrom = child.extends
  if (extendsFrom) {
    parent = mergeOptions(parent, extendsFrom, vm)
  }
  // mixin是选项的集合，调用mergeOptions合并选项
  if (child.mixins) {
    for (let i = 0, l = child.mixins.length; i < l; i++) {
      parent = mergeOptions(parent, child.mixins[i], vm)
    }
  }
  const options = {}
  let key
  // merge parent上的选项
  for (key in parent) {
    mergeField(key)
  }
  // merge child独有的选项
  for (key in child) {
    if (!hasOwn(parent, key)) {
      mergeField(key)
    }
  }
  // 选择不同的merge策略合并选项参数
  function mergeField (key) {
    const strat = strats[key] || defaultStrat
    options[key] = strat(parent[key], child[key], vm, key)
  }
  return options
}
```