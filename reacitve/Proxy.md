# Proxy

语法：

```javascript
let proxy = new Proxy(target, handler)
```

- target: 要包装的对象，可以是任何类型的对象，包括函数
- handler： 代理配置项，定义拦截行为

如果在 handler 中定义了相应操作的拦截方法，当操作时，proxy 会执行相应的拦截方法，否则会对 target 进行相应的操作。

```javascript
let target = {}
let proxu = new Proxu(target, {})

proxy.test = 5
console.log(target.test) // 5
console.log(proxy.test) // 5
```

_Proxy_ 自身没有任何属性，如果 _handler_ 是空的，所有操作会直接转发到 _target_。

_handler_ 参数

| Internal Method         | Handler Method             | Triggers when…                                               |
| :---------------------- | :------------------------- | :----------------------------------------------------------- |
| `[[Get]]`               | `get`                      | reading a property                                           |
| `[[Set]]`               | `set`                      | writing to a property                                        |
| `[[HasProperty]]`       | `has`                      | `in` operator                                                |
| `[[Delete]]`            | `deleteProperty`           | `delete` operator                                            |
| `[[Call]]`              | `apply`                    | function call                                                |
| `[[Construct]]`         | `construct`                | `new` operator                                               |
| `[[GetPrototypeOf]]`    | `getPrototypeOf`           | [Object.getPrototypeOf](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getPrototypeOf) |
| `[[SetPrototypeOf]]`    | `setPrototypeOf`           | [Object.setPrototypeOf](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf) |
| `[[IsExtensible]]`      | `isExtensible`             | [Object.isExtensible](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/isExtensible) |
| `[[PreventExtensions]]` | `preventExtensions`        | [Object.preventExtensions](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/preventExtensions) |
| `[[DefineOwnProperty]]` | `defineProperty`           | [Object.defineProperty](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty), [Object.defineProperties](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperties) |
| `[[GetOwnProperty]]`    | `getOwnPropertyDescriptor` | [Object.getOwnPropertyDescriptor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyDescriptor), `for..in`, `Object.keys/values/entries` |
| `[[OwnPropertyKeys]]`   | `ownKeys`                  | [Object.getOwnPropertyNames](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertyNames), [Object.getOwnPropertySymbols](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getOwnPropertySymbols), `for..in`, `Object.keys/values/entries` |



##### 参考资料

[Proxy and Reflect](https://javascript.info/proxy)

