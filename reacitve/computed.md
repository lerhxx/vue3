# computed

计算属性。对于任何包含响应式数据的复杂逻辑，都应使用 computed。

### 使用方法

和 Vue2 一样，可以接受 `getter` 函数，或者带有 `get` 和 `set` 函数的对象，返回一个不可变的响应式 ref 对象。

getter 函数没有副作用。

##### 创建

```javascript
const count = ref(1)
const plusOne = computed(() => count.value + 1)
```

```javascript
const count = ref
const plusOne = computed({
  get: () => count.value. + 1,
  set: value => {
    count.value = value
  }
})
```

##### 读取

```
console.log(plusOne.value) // 2
```

computed 的值需要从 value 属性中获取。

### 特性

计算属性可能会依赖其他响应式状态，同时会延迟和缓存计算值。

### 问题

1. 为什么需要通过 value 属性获取 computed 的值
2. 缓存怎么实现？？
3. 哪些地方会延迟计算？
4. 什么时候才会计算 getter？
5. Vue 如何知道需要收集哪些依赖？



##### 问题一

```
console.log(plusOne) // ComputedRefImpl{...}
```

输出 plusOne，得到的是一个 ComputedRefImpl 的实例。从源码可以发现，读取 value 时会调用 track 去收集依赖。

也就是说只有访问 value 属性才能达到收集依赖的目的。

##### 问题二

缓存借助了一个私有属性 _dirty，来控制是否需要重新计数 getter，并将结果保存在另一个私有属性 _value 中。

_dirty 初始值为 true，第一次访问 .value 时，会计算 getter 赋值给 _value，并将 _dirty 设置为 false。在依赖更新之前，再次读取 .value 都会直接返回 _value，并重新收集依赖。

##### 问题三

* 创建时，不会立即计算 getter
* 依赖的状态更新时，不会立即计算 getter

##### 问题四

先来看看 constructor 中的 this.effect，得到的是一个包裹了 getter 的函数，当调用 this.effect 时，内容会执行 getter，并将值返回。也就是说执行 this.effect 相当于执行 getter。

执行 constructor 时，整个过程只是创建了 this.effect，但没用调用。

再来看看，当依赖的响应式状态更新时，会调用创建 this.effect 时传入的 scheduler，而 scheduler 内部也同样没有调用 this.effect，只是将 _dirty 设置为 true，并通知它的被依赖项更新。

因此，computed 创建、依赖更新时都不会立即计算 getter，这就是 computed 的延迟特性。

整个 Class 中只有 get value 中调用了 this.effect。不管是创建之后还是依赖更新之后，都需要读取 computed.value ，才会执行 this.effect（getter），得到新的值。

##### 问题五

Vue3 中，必须通过 track 进行依赖的收集。响应式状态都会在执行 Get 操作时调用 track 来收集依赖。

对于computed，Vue 不需要知道 getter 内部依赖了哪些状态，computed 会告诉 Vue 它依赖的状态。什么时候会告诉呢？

看看 get value 就知道，当依赖 computed 的地方，读取 computed.value 时，就会调用 track，每次都会调用，这就能完成了依赖的收集。

[源码](https://github.com/vuejs/vue-next/blob/5d825f318f1c3467dd530e43b09040d9f8793cce/packages/reactivity/src/computed.ts)

```javascript
class ComputedRefImpl<T> {
  private _value!: T
  private _dirty = true

  public readonly effect: ReactiveEffect<T>

  public readonly __v_isRef = true;
  public readonly [ReactiveFlags.IS_READONLY]: boolean

  constructor(
    getter: ComputedGetter<T>,
    private readonly _setter: ComputedSetter<T>,
    isReadonly: boolean
  ) {
    this.effect = effect(getter, {
      lazy: true,
      scheduler: () => {
        if (!this._dirty) {
          this._dirty = true
          trigger(toRaw(this), TriggerOpTypes.SET, 'value')
        }
      }
    })

    this[ReactiveFlags.IS_READONLY] = isReadonly
  }

  get value() {
    if (this._dirty) {
      this._value = this.effect()
      this._dirty = false
    }
    track(toRaw(this), TrackOpTypes.GET, 'value')
    return this._value
  }

  set value(newValue: T) {
    this._setter(newValue)
  }
}
```











