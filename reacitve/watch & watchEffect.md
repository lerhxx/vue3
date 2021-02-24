# watch & watchEffect

定义

```javascript
export interface WatchOptionsBase {
  flush?: 'pre' | 'post' | 'sync'
  onTrack?: ReactiveEffectOptions['onTrack']
  onTrigger?: ReactiveEffectOptions['onTrigger']
}

// Simple effect.
export function watchEffect(
  effect: WatchEffect,
  options?: WatchOptionsBase
): WatchStopHandle {
  return doWatch(effect, null, options)
}

```

使用 🌰

```javascript
const count = ref(0)

const stop = watchEffect(() => console.log(count.value))
```

传入的函数在创建时会`立即执行`，执行函数的时候会进行依赖收集；

当依赖更新时，会运行传入的函数。

如果 watchEffect 是在 setup 或者生命周期钩子中调用的，那么侦听器会被挂载到组件中。

### 停止监听

调用 watchEffect 会得到一个函数，用于停止监听。

```javascript
function dowatch() {
	...
	return () => {
    stop(runner)
    if (instance) {
      remove(instance.effects!, runner)
    }
  }
}
```

这个函数负责两件事

* 清除副作用，如何清除是由开发者定义的
* 如果被侦听器挂载到组件，就从组件中卸载



