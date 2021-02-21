# reactive

```javascript
const a = reactive({ count; 0 })
```

* reactive 是基于 Proxy 实现的，只能接受对象作为参数，返回对象的响应式代理。
* 代理对象不等于原对象。
* reactive 等同于 2.x 的 `Vue.observable()`。
* 转换时 *深层的*，会影响对象内部所有嵌套的属性。

#### reactive 方法的定义

```javascript
export function reactive<T extends object>(target: T): UnwrapNestedRefs<T>
export function reactive(target: object) {
  // if trying to observe a readonly proxy, return the readonly version.
  // 如果 target 只读，则直接返回 target
  if (target && (target as Target)[ReactiveFlags.IS_READONLY]) {
    return target
  }
  return createReactiveObject(
    target,
    false,
    mutableHandlers,
    mutableCollectionHandlers
  )
}
```

## createReactiveObject

createReactiveObject 负责：

* 处理 target 是否可被观察，是则直接返回 target
* 处理 target 是否已被观察，是则返回被观察的对象

最终返回 target 的代理对象

对于 proxy，最重要的就是 handler 的逻辑，在 reactive 中，对于不同类型的对象，用了不同的 handler 进行代理。



*****



## Handler

重点来啦。

![img](https://github.com/lerhxx/vue3/blob/master/images/reactive.png)

从图中可以很清楚的看到，不管是什么类型的 target，最终的目的都一样：

* 获取属性时通过 effect 提供的 track 收集依赖
* 更新、删除属性时通过 effect 提供的 trigger 触发依赖更新

而所有的依赖都被保存在 targetMap 中。

先来看看可读 Array/Object 的 handler。

```javascript
// ba seHandler
export const mutableHandlers: ProxyHandler<object> = {
  get,
  set,
  deleteProperty,
  has,
  ownKeys
}
```

#### Get

在 get 的逻辑中，对 Array 某些方法做了特殊处理。并对类型为 Object 的属性值做了延迟访问。

```javascript
function get(target: Target, key: string | symbol, receiver: object) {
  if (key === ReactiveFlags.IS_REACTIVE) {
    return !isReadonly
  } else if (key === ReactiveFlags.IS_READONLY) {
    return isReadonly
  } else if (
    key === ReactiveFlags.RAW &&
    receiver === (isReadonly ? readonlyMap : reactiveMap).get(target)
  ) {
    return target
  }

  const targetIsArray = isArray(target)
  
  // target 为 Array，且 key 为 [includes, indexOf, push, pop, shift, unshift, splice] 中的一个，
  // 则以 arrayInstrumentations 作为目标对象，获取 key 值
  // arrayInstrumentations 是对几个数组方法的封装，具体内容先跳过，晚点再看
  if (!isReadonly && targetIsArray && hasOwn(arrayInstrumentations, key)) {
    return Reflect.get(arrayInstrumentations, key, receiver)
  }

  // 这里就得到了 target[key] 值
  const res = Reflect.get(target, key, receiver)
  
  if (
    isSymbol(key)
      ? builtInSymbols.has(key as symbol)
      : key === `__proto__` || key === `__v_isRef`
  ) {
    return res
  }

  if (!isReadonly) {
    // 收集依赖
    track(target, TrackOpTypes.GET, key)
  }

  if (shallow) {
    return res
  }

  if (isRef(res)) {
    // ref unwrapping - does not apply for Array + integer key.
    const shouldUnwrap = !targetIsArray || !isIntegerKey(key)
    return shouldUnwrap ? res.value : res
  }
  
  if (isObject(res)) {
    // Convert returned value into a proxy as well. we do the isObject check
    // here to avoid invalid value warning. Also need to lazy access readonly
    // and reactive here to avoid circular dependency.
    // 如果 target[key] 是对象，会返回一个代理对象。检查是否 Object 是为了避免无效值的警告。
    // 延迟访问 readonly 和 reactive 是为了避免循环依赖。
    return isReadonly ? readonly(res) : reactive(res)
  }

  return res
}
```

现在来看看 arrayInstrumentations。

```java
const arrayInstrumentations: Record<string, Function> = {}
// instrument identity-sensitive Array methods to account for possible reactive
// value
;(['includes', 'indexOf', 'lastIndexOf'] as const).forEach(key => {
  const method = Array.prototype[key] as any
  arrayInstrumentations[key] = function(this: unknown[], ...args: unknown[]) {
    const arr = toRaw(this)
    for (let i = 0, l = this.length; i < l; i++) {
      track(arr, TrackOpTypes.GET, i + '')
    }
    // we run the method using the original args first (which may be reactive)
    const res = method.apply(arr, args)
    if (res === -1 || res === false) {
      // if that didn't work, run it again using raw values.
      return method.apply(arr, args.map(toRaw))
    } else {
      return res
    }
  }
})
// instrument length-altering mutation methods to avoid length being tracked
// which leads to infinite loops in some cases (#2137)
// 对会改变数组长度的方法，另外封装，不收集依赖，为了避免某些情况可能导致的无限循环
;(['push', 'pop', 'shift', 'unshift', 'splice'] as const).forEach(key => {
  const method = Array.prototype[key] as any
  arrayInstrumentations[key] = function(this: unknown[], ...args: unknown[]) {
    // 不允许收集依赖
    pauseTracking()
    const res = method.apply(this, args)
    resetTracking()
    return res
  }
})
```

#### Set

Set 的逻辑只要是区分 add 和 set，并返回更新结果。

```javascript
return function set(
  target: object,
  key: string | symbol,
  value: unknown,
  receiver: object
): boolean {
  const oldValue = (target as any)[key]
  if (!shallow) {
    value = toRaw(value)
    if (!isArray(target) && isRef(oldValue) && !isRef(value)) {
      oldValue.value = value
      return true
    }
  } else {
    // in shallow mode, objects are set as-is regardless of reactive or not
  }

  const hadKey =
    isArray(target) && isIntegerKey(key)
      ? Number(key) < target.length
      : hasOwn(target, key)
  const result = Reflect.set(target, key, value, receiver)
  // don't trigger if target is something up in the prototype chain of original
  if (target === toRaw(receiver)) {
    if (!hadKey) {
      trigger(target, TriggerOpTypes.ADD, key, value)
    } else if (hasChanged(value, oldValue)) {
      trigger(target, TriggerOpTypes.SET, key, value, oldValue)
    }
  }
  return result
}
```

#### DeleteProperty

删除成功会调用 trigger 通知被依赖项更新，并返回结果。

#### Has、OwnKeys

这两个方法调用 track 收集依赖，并返回 Reflect 对应操作的结果