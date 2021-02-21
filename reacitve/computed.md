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



#### 为什么需要通过 value 属性获取 computed 的值？

```
console.log(plusOne) // ComputedRefImpl{...}
```

输出 plusOne，得到的是一个 ComputedRefImpl 的实例。从源码可以发现，读取 value 时会调用 track 去收集依赖。

也就是说只有访问 value 属性才能达到收集依赖的目的。



#### 缓存怎么实现？

缓存借助了一个私有属性 _dirty，来控制是否需要重新计数 getter，并将结果保存在另一个私有属性 _value 中。

_dirty 初始值为 true，

访问 .value 时，

* 如果_dirty 为 true
  * 计算 getter 并赋值给  _value
  * 将 _dirty 设置为 false；
* 触发 track ，computed 作为依赖被收集。

当 computed 的依赖更新触发 trigger，执行 computed 的 scheduler。scheduler 中，只有当  _dirty 为 `false` 时，会负责两件事：

* 将 _dirty 置为 `true`
* 触发 trigger，更新 computed 的被依赖项

在依赖更新之前，再次读取 .value 都会直接返回 _value，并重新收集依赖。



#### 哪些时候会延迟计算？

* 创建时，不会立即计算 getter
* 依赖的状态更新时，不会立即计算 getter

以下例子，getter 会被执行多少次，点击 button 的时候 getter 会执行吗

```javascript
// template
<div>
  <div>{{count}}</div>
  <button @click="add">Add Count</button>
</div>

// setup()
const count = ref<number>(0)
const plusOne = computed(() => {
  console.log('plusOne')
  return count.value + 1
})

function add() {
  ++count.value
}
```

运行之后，'plusOne' 没有输出过。

***computed `创建时不会`计算 getter。***

1. 什么时候才计算 getter 呢？

   看源码能发现，只有读取 `value` 且 \_dirty 为 true 时才会计算 getter。🌰 中没有读取过 plusOne.value。

2. 为什么 count 更新时，也没计算 getter 呢？

   **在 Vue3 中，只有调用了 `track()` 才能进行依赖的收集。**

   在 ComputedRefImpl 中，也是只有在 `value` 中才会调用 `track`。也就是说， 只有读取过 plusOne.value 才能收集到依赖。🌰 中没有读取过plusOne.value。

   count `没有`被 plusOne 收集到，那么当 count 更新时，plusOne 也就不会接收到通知了。

```javascript
// setup()
console.log('console', plusOne.value)
```

在 setup 中加上一句 console.log，'plusOne' 输出了，但是更新 count，依然不会输出 'plusOne'，为什么呢？

🌰 中已经读取过 plusOne.value，plusOne 已经对 count 进行了依赖收集。当 count 更新时，plusOne 就能够接到更新通知。

当 plusOne 更新时，就调用 scheduler。scheduler 只负责两件事，当 \_dirty 为 `false` 时：

* _dirty 设置为 `true`
* 调用 `trigger`，通知 plusOne 的被依赖项更新

*** computed `更新时不会`执行getter ***

count 更新时，真的通知到 computed 了吗？

```javascript
// setup
watch(count, () => {
  console.log('watch', plusOne)
})
```

留意 plusOne 的 `_dirty` 值，count 更新之后，\_dirty 变成 true 了。

只有 *scheduler* 会将 \_dirty 设置为 true，也就可以证明当 computed 的依赖更新时，执行的时 scheduler。

*什么时候才会计算 getter 呢？*

看看前文。

当 console.log 执行过后，没有其它地方读取 plusOne.value。所以即使 plusOne 的依赖 count 更新了，也不会去计算 getter。

```javascript
// template
<div class="dom">{{plusOne}}</div>
```

在 template 中加上 plusOne 的引用，当 count 更新的同时，控制台也会输出 'plusOne'。

就像前面提到的，plusOne 更新时，并不会重新计算 getter 。🌰 中，是在生成 Virtual DOM，解析到 .dom 的值时，才会计算 getter。



#### 什么时候才会计算 getter？

先来看看 constructor 中的 this.effect，得到的是一个包裹了 getter 的函数，当调用 this.effect 时，内容会执行 getter，并将值返回。也就是说执行 this.effect 相当于执行 getter。

执行 constructor 时，整个过程只是创建了 this.effect，但没用调用。

再来看看，当依赖的响应式状态更新时，会调用创建 this.effect 时传入的 scheduler，而 scheduler 内部也同样没有调用 this.effect，只是将 _dirty 设置为 true，并通知它的被依赖项更新。

因此，computed 创建、依赖更新时都不会立即计算 getter，这就是 computed 的延迟特性。

整个 Class 中只有 get value 中调用了 this.effect。不管是创建之后还是依赖更新之后，都需要读取 computed.value ，才会执行 this.effect（getter），得到新的值。



#### Vue 怎么知道需要收集哪些依赖？

Vue3 中，必须通过 track 进行依赖的收集。响应式状态都会在执行 Get 操作时调用 track 来收集依赖。

对于computed，Vue 不需要知道 getter 内部依赖了哪些状态，computed 会告诉 Vue 它依赖的状态。什么时候会告诉呢？

看看 get value 就知道，当依赖 computed 的地方，读取 computed.value 时，就会调用 track，每次都会调用，这就能完成了依赖的收集。



---



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
      // 当依赖更新时，会执行 scheduler，而不是 this.effect
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











