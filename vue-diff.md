# 跟着vue源码学diff

## 模板方法注册
  core/instance/index.js （renderMixin） ---》 core/instance/render.js(renderMixin) 里面执行方法installRenderHelpers(Vue.prototype)
  core/instance/render-helper/index.js
```
export function installRenderHelpers (target: any) {
  target._o = markOnce
  target._n = toNumber
  target._s = toString
  target._l = renderList
  target._t = renderSlot
  target._q = looseEqual
  target._i = looseIndexOf
  target._m = renderStatic
  target._f = resolveFilter
  target._k = checkKeyCodes
  target._b = bindObjectProps
  target._v = createTextVNode
  target._e = createEmptyVNode
  target._u = resolveScopedSlots
  target._g = bindObjectListeners
  target._d = bindDynamicKeys
  target._p = prependModifier
}
```
模板编译之后如下，里面含有_c,_s等对应上的方法。

```
(function() {
    with (this) {
        return _c('div', {
            attrs: {
                "id": "app"
            }
        }, [_c('p', [_v(_s(name))]), _v(" "), _c('p', {
            domProps: {
                "textContent": _s(name)
            }
        }), _v(" "), _c('p', [_v(_s(age))]), _v(" "), _c('input', {
            directives: [{
                name: "model",
                rawName: "v-model",
                value: (name),
                expression: "name"
            }],
            attrs: {
                "type": "text"
            },
            domProps: {
                "value": (name)
            },
            on: {
                "input": function($event) {
                    if ($event.target.composing)
                        return;
                    name = $event.target.value
                }
            }
        }), _v(" "), _c('mycomponent', {
            attrs: {
                "count": count,
                "age": age
            }
        })], 1)
    }
}
```
## 数据变更之后，创建新的vnode，然后和老的比较
在core/instance/lifecircle.js中的mountComponent方法，即页面mount时执行的。
传进去的是updateComonent，Wather里面会去执行updateComponent(this.getter.call()),里面执行了vm._render，所以会去执行绑定的数据的get方法。
```
updateComponent = () => {
      vm._update(vm._render(), hydrating)
    }
```
页面Mount时，会new一个watcher，在watcher的构造方法里面
```
new Watcher(vm, updateComponent, noop, {
    before () {
      if (vm._isMounted && !vm._isDestroyed) {
        callHook(vm, 'beforeUpdate')
      }
    }
  }, true /* isRenderWatcher */)
```
在数据绑定的时候，定义的get，会看是否有Dep.target，那么在上面new watcher的时候，会pushTarget当前实例，然后执行依赖的depend，给当前绑定的对象的new Dep(),增加一个watcher。
数据Data/prop等需要绑定的具体变量----》new 一个唯一的Dep依赖----》Dep可以push很多watcher
```
Object.defineProperty(obj, key, {
    enumerable: true,
    configurable: true,
    get: function reactiveGetter () {
      const value = getter ? getter.call(obj) : val
      if (Dep.target) {
        dep.depend()
        if (childOb) {
          childOb.dep.depend()
          if (Array.isArray(value)) {
            dependArray(value)
          }
        }
      }
})
……
depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }
```
每当Dep有变更，会通知Watcher，然后watcher会去触发更新，执行update,进入core/observe/scheduler/queueWatcher,然后触发nextTick，执行watcher的run，最终回调到updsateComponent，执行新老vnode比较。

## VNode比较。
updateComponent方法首先获取新的VNode，然后执行_update。获取老的VNode，然后将新的VNode赋值给vm._vnode，然后执行__patch__(src/platforms/runtime/index.js),执行core/vdom/patch，正式执行VNode比较。

- patchNode(core/vdom/patch.js)
```
const oldCh = oldVnode.children
const ch = vnode.children
```
if 新节点没有text
 - [ ] 看新老节点是否有子节点，如果都有子节点，但是不相等，则updateChildren
 - [ ] 如果新节点有值ch（即老节点没子节点），则直接addVnodes
 - [ ] 如果老节点有值oldCh（即新节点没子节点），则直接removeVnodes
 - [ ] 如果老节点有值（新节点没值）,则清空值
 
else 新节点有text,则直接      nodeOps.setTextContent(elm, vnode.text)

- addVnodes
将新节点的数据，一一创建节点，更新Dom
```
function addVnodes (parentElm, refElm, vnodes, startIdx, endIdx, insertedVnodeQueue) {
    for (; startIdx <= endIdx; ++startIdx) {
      createElm(vnodes[startIdx], insertedVnodeQueue, parentElm, refElm, false, vnodes, startIdx)
    }
  }
```
- removeVnodes
将老节点全部删除

- updateChildren
1、如果没有oldStartVnode，则前移一步oldStartIdx
2、如果没有oldEndVnode，则退后一步oldEndIdx
以上两步保证oldVnode两边都是有节点
3、如果新老起始节点相同，则执行patchVnode，更新节点。然后oldStartIdx+1，newStartIdx+1
4、如果新老末尾节点相同，则执行patchVnode,更新节点，然后oldEndIdx-1，newEndIdx-1
5、如果老的起始节点oldStartVnode和新的newEndVnode相同，则更新节点内容，然后将oldStartVnode插入到oldEndVnode之后，然后oldStartIdx+1，newEndIdx-1.
6、如果老的末尾节点和新的起始节点相同，则更新节点内容，则更新内容，然后将oldEndVnode节点插入到oldStartVnode节点之前。oldEndIdx-1，newStartIdx+1。
7、以上都没匹配上，
   a) 把老的节点都创建key的map对象。
   b) 在老的节点中找到新的节点的对应索引：如果新的有key，老的无key，则没找到，如果新老都没有key，在循环老节点，去sameNode找到新node对应的key.
   c) 如果没有找到对应的老节点，则新建一个节点，插入到老节点之前
   d) 如果找到了对应的老节点且是sameNode，则更新老节点内容，将找到的索引的老节点移动到当前老节点之前。
   f) 如果找到的老节点，不是sameNode，则新建节点，插入到当前老节点之前（？为啥不在找对应老节点的时候，使用sameNode？）
   g) newStartIdx++
   
8、执行1-7循环，知道while截止，
 如果oldStartIdx > oldEndIdx,说明匹配规则，在移动老节点，老节点都处理完成了，则需要把剩下的新节点newStartIdx 到 newEndIdx 新建放到oldStartIdx之前
 如果newStartIdx > newEndIdx，说明新节点都处理完成，剩下的有些老节点没有处理，则需要删除剩下的老节点oladStartIdx 到 oldEndIdx的节点。
