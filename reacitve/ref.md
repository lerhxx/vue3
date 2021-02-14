# ref

接受一个参数值并返回一个响应式且可改变的 ref 对象。ref 对象拥有一个指向内部值的单一属性 `.value`.

```javascript
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

ref 的内部值在以下几种情况下会自动解套，不需要通过 .value 访问：

* 在`模版`中使用时
* 作为 `reactive` 对象 property 被访问或修改时

#### ref 和 reactive 有什么区别？

reactive：内部使用的是 `Proxy` 实现，Proxy 只能接受对象作为入参，所以 reactive 也只能接受`对象`作为入参。

ref：可以即接受`原始值`作为入参，也可以接受`对象`作为入参。当入参是对象时，内部会调用 reactive 进行深层转换。



### 为什么需要通过 .value 访问呢？

```javascript
console.log(count)
```

将 count 输出可以发现，count 是一个实例。

来看看 ref 的实现。

```javascript
export function ref(value?: unknown) {
  return createRef(value)
}

function createRef(rawValue: unknown, shallow = false) {
  if (isRef(rawValue)) {
    return rawValue
  }
  return new RefImpl(rawValue, shallow)
}


class RefImpl<T> {
  private _value: T

  public readonly __v_isRef = true

  constructor(private _rawValue: T, public readonly _shallow = false) {
    this._value = _shallow ? _rawValue : convert(_rawValue)
  }

  get value() {
    track(toRaw(this), TrackOpTypes.GET, 'value')
    return this._value
  }

  set value(newVal) {
    if (hasChanged(toRaw(newVal), this._rawValue)) {
      this._rawValue = newVal
      this._value = this._shallow ? newVal : convert(newVal)
      trigger(toRaw(this), TriggerOpTypes.SET, 'value', newVal)
    }
  }
}

const convert = <T extends unknown>(val: T): T =>
  isObject(val) ? reactive(val) : val
```

从 RefImpl 中可以看出，只有访问 ref 的 value 属性，才能触发 track 收集依赖，更新 value 时，会触发 trigger 通知被依赖项更新。

从 constructor 可以知道，ref 的入参保存在了私有属性 _value 中，当入参是对象是，会保存经过 reacitve 转换后的代理对象。

