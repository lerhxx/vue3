# computed

[源码](https://github.com/vuejs/vue-next/blob/5d825f318f1c3467dd530e43b09040d9f8793cce/packages/reactivity/src/computed.ts)

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



##### 问题一

```
console.log(plusOne) // ComputedRefImpl{...}
```

输出 plusOne，得到的是一个 ComputedRefImpl 的实例









