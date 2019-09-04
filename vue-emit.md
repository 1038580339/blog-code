### vue emit

本文接上一篇 [vue组件通信方式](vue-prop.md)

对于理解emit来说，我决定还是从源码入手

至于下面我是如何找到这个文件的，请查看 [Vue源码解析](https://www.jianshu.com/p/557c8c1c250f)  

ok，进入正题

此时我们来到了 \vue\src\core\instance\events.js

接下来找到我们这次需要的主角

```javascript
Vue.prototype.$emit = function (event: string): Component {
    const vm: Component = this
    if (process.env.NODE_ENV !== 'production') {
      const lowerCaseEvent = event.toLowerCase()
      if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
        tip(
          `Event "${lowerCaseEvent}" is emitted in component ` +
          `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
          `Note that HTML attributes are case-insensitive and you cannot use ` +
          `v-on to listen to camelCase events when using in-DOM templates. ` +
          `You should probably use "${hyphenate(event)}" instead of "${event}".`
        )
      }
    }
    let cbs = vm._events[event]
    if (cbs) {
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
      const args = toArray(arguments, 1)
      const info = `event handler for "${event}"`
      for (let i = 0, l = cbs.length; i < l; i++) {
        invokeWithErrorHandling(cbs[i], vm, args, vm, info) // handler: Function,context: any,args: null | any[],vm: any,info: string
      }
    }
    return vm
  }
```

##### 此处，在Vue.prototype上定义了$emit,接受参数(event:string) 返回 component

接下来分析下内部函数

```javascript
      if (process.env.NODE_ENV !== 'production') {
      const lowerCaseEvent = event.toLowerCase()
      if (lowerCaseEvent !== event && vm._events[lowerCaseEvent]) {
        tip(
          `Event "${lowerCaseEvent}" is emitted in component ` +
          `${formatComponentName(vm)} but the handler is registered for "${event}". ` +
          `Note that HTML attributes are case-insensitive and you cannot use ` +
          `v-on to listen to camelCase events when using in-DOM templates. ` +
          `You should probably use "${hyphenate(event)}" instead of "${event}".`
        )
      }
    }
```

这一整段作为生产环境下的提示，与实现无关，我们忽略

```javascrfipt
    let cbs = vm._events[event]
```
这里作为一开始执行的第一句，event是我们传入参数，vm._events又是什么东西呢？

```javascript
export function initEvents (vm: Component) {
  vm._events = Object.create(null)
  vm._hasHookEvent = false
  // init parent attached events
  const listeners = vm.$options._parentListeners
  if (listeners) {
    updateComponentListeners(vm, listeners)
  }
}
```
##### _events在文件中暴露的initevents中定义

这里还需看下$on

```javascript
     Vue.prototype.$on = function (event: string | Array<string>, fn: Function): Component {
    const vm: Component = this
    if (Array.isArray(event)) {
      for (let i = 0, l = event.length; i < l; i++) {
        vm.$on(event[i], fn)
      }
    } else {
      (vm._events[event] || (vm._events[event] = [])).push(fn)
      // optimize hook:event cost by using a boolean flag marked at registration
      // instead of a hash lookup
      if (hookRE.test(event)) {
        vm._hasHookEvent = true
      }
    }
    return vm
  }
```
##### $on时，将事件挂到了_events上

到这里，我们基本知道了_event的来源，那么我们到下一段

```javavscript
let cbs = vm._events[event]
if (cbs) {
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
      const args = toArray(arguments, 1)
      const info = `event handler for "${event}"`
      for (let i = 0, l = cbs.length; i < l; i++) {
        invokeWithErrorHandling(cbs[i], vm, args, vm, info) // handler: Function,context: any,args: null | any[],vm: any,info: string
      }
    }
```
如果这个事件存在于_event时，则进行下面操作，那么toArray是什么？

```javascript
import {
  tip,
  toArray,
  hyphenate,
  formatComponentName,
  invokeWithErrorHandling
} from '../util/index'
```

从../util/index引入，于是我找到了这个文件，于是发生了一件有点尴尬的事
```javavscript
export * from 'shared/util'
export * from './lang'
export * from './env'
export * from './options'
export * from './debug'
export * from './props'
export * from './error'
export * from './next-tick'
export { defineReactive } from '../observer/index'
```

长成这个我怎么找？？？ 不过总是能找到的嘛

```javascript
/**
 * Convert an Array-like object to a real Array.
 */
export function toArray (list: any, start?: number): Array<any> {
  start = start || 0
  let i = list.length - start
  const ret: Array<any> = new Array(i)
  while (i--) {
    ret[i] = list[i + start]
  }
  return ret
}

```

由上可见，该函数主要用于将list转为数组，以及可以选择从数组哪里开始

ok,我们返回源码

```javavscript
let cbs = vm._events[event]
if (cbs) {
      cbs = cbs.length > 1 ? toArray(cbs) : cbs
      const args = toArray(arguments, 1)
      const info = `event handler for "${event}"`
      for (let i = 0, l = cbs.length; i < l; i++) {
        invokeWithErrorHandling(cbs[i], vm, args, vm, info) // handler: Function,context: any,args: null | any[],vm: any,info: string
      }
    }
```
这个时候就比较明确了，如果是cbs如果>1,转为数组。args转为数组，并从第一个开始，为什么要从第一个开始呢？？？

这个就要从Vue官方文档说起了，$emit有两种用法 [emit用法](https://cn.vuejs.org/v2/api/#vm-emit)

```javascript
vm.$emit( eventName, […args] )

this.$emit('welcome')

this.$emit('give-advice', this.possibleAdvice[randomAdviceIndex])
```

#### emit第一参数作为事件名传入，余下参数作为事件函数参数传入

最后，我们来到最后一个函数

```javavscript
    invokeWithErrorHandling(cbs[i], vm, args, vm, info) // handler: Function,context: any,args: null | any[],vm: any,info: string 
```

这里，我已经备注了这几个参数名，我们看到这个函数

```javascript
export function invokeWithErrorHandling (
  handler: Function,
  context: any,
  args: null | any[],
  vm: any,
  info: string
) {
  let res
  try {
    res = args ? handler.apply(context, args) : handler.call(context)
    if (res && !res._isVue && isPromise(res) && !res._handled) {
      res.catch(e => handleError(e, vm, info + ` (Promise/async)`))
      // issue #9511
      // avoid catch triggering multiple times when nested calls
      res._handled = true
    }
  } catch (e) {
    handleError(e, vm, info)
  }
  return res
}
```
try中为函数执行

```javascript
res = args ? handler.apply(context, args) : handler.call(context)
```

第一句如果args存在，作为参数传入，这个刚才说过了emit的使用方式，这边就体现了为什么要toArray(args,1)去掉第一个参数。

```javascript
if (res && !res._isVue && isPromise(res) && !res._handled) {
      res.catch(e => handleError(e, vm, info + ` (Promise/async)`))
      // issue #9511
      // avoid catch triggering multiple times when nested calls
      res._handled = true
    }
```

这边是对res函数出错的捕获报错，res存在判断，是否是promise判断，是否没有处理过，那_isVue是什么？？

这里我们继续看到Vue的_init方法

```javascript
    export function initMixin (Vue: Class<Component>) {
  Vue.prototype._init = function (options?: Object) {
    const vm: Component = this
    // a uid
    vm._uid = uid++

    let startTag, endTag
    /* istanbul ignore if */
    if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
      startTag = `vue-perf-start:${vm._uid}`
      endTag = `vue-perf-end:${vm._uid}`
      mark(startTag)
    }

    // a flag to avoid this being observed
    vm._isVue = true
    // merge options
    下面是混合options
}
```

_init时有一句注释，

##### a flag to avoid this being observed  避免被观察到的标志

也就是说如果是避免被观察到的。。。我就不管他了，不去catch了。


ok，本文到现在就结束了


***

作者：  呼啦啦圈圈圆圆年年  

本文纯原创。。。









