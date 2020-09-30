## mutableHandlers
    当target是对象或者数组时, 对proxy进行配置,
```js
    const mutableHandlers: ProxyHandler<object> = {
        get, // createGetter()
        set,
        deleteProperty,
        has,
        ownKeys
    }

```


## createGetter
```js
    // 这里是 reactive 的 get 代理方法
    // 一个常规Proxy的配置
    function createGetter(isReadonly = false, shallow = false) {
        // target : 目标函数; key: 目标函数的属性值; 
        // receiver: 它总是指向原始的读操作所在的那个对象，一般情况下就是 Proxy 实例
        return function get(target: Target, key: string | symbol, receiver: object) {
            // 对标记对象做特殊处理
            if (key === ReactiveFlags.IS_REACTIVE) { // 如果 key === __v_isReactive
                return !isReadonly
            } else if (key === ReactiveFlags.IS_READONLY) { // 如果 key === __v_isReadonly
                return isReadonly
            } else if (
                key === ReactiveFlags.RAW &&
                receiver === (isReadonly ? readonlyMap : reactiveMap).get(target)
            ) { // 如果 key === __v_raw && receiver 已经在 Map 中 则表明 target 已经被代理,直接返回
                return target
            }

            const targetIsArray = isArray(target)
            // [FX] 如果是个数组,并且 arrayInstrumentations (参照下面的) 拥有对应key
            // 主要是劫持数组的一些特殊方法 做加工后返回给实际对象使用
            if (targetIsArray && hasOwn(arrayInstrumentations, key)) {
                return Reflect.get(arrayInstrumentations, key, receiver)
            }

            // 获取 target[key]
            const res = Reflect.get(target, key, receiver)

            // 判断是否是 Symbol 对象
            // builtInSymbols 作用: 获取 Symbol 对象的所有 key 对应的值, 然后 勇 isSymbol 过滤;得到是 Symbol 对象 的值 生成一个 set 对象
            // 如果key 是一个  Symbol 对象 或者 是原型/ref 则直接返回 res 值
            const keyIsSymbol = isSymbol(key)
            if (
                keyIsSymbol
                    ? builtInSymbols.has(key as symbol)
                    : key === `__proto__` || key === `__v_isRef`
            ) {
                return res
            }

            // 如果不是 只读
            if (!isReadonly) {
                // 直接对目标进行 track 追踪
                track(target, TrackOpTypes.GET, key)
            }

            // 肤浅模式 直接返回 若不是,则将 res 对象 进行深度追踪
            if (shallow) {
                return res
            }

            // 这里是将 ref 直接展开
            if (isRef(res)) {
                // ref unwrapping - does not apply for Array + integer key.
                // 不适用数组+整数键。
                const shouldUnwrap = !targetIsArray || !isIntegerKey(key)
                return shouldUnwrap ? res.value : res
            }

            // 如果 res 是一个 对象
            if (isObject(res)) {
                // Convert returned value into a proxy as well. we do the isObject check
                // here to avoid invalid value warning. Also need to lazy access readonly
                // and reactive here to avoid circular dependency.
                // 递归执行
                return isReadonly ? readonly(res) : reactive(res)
            }

            return res
        }
    }
```

## arrayInstrumentations
    [FX 作用?]
    劫持了一系列数组方法
```js
const arrayInstrumentations: Record<string, Function> = {}
// instrument identity-sensitive Array methods to account for possible reactive
// values
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
;(['push', 'pop', 'shift', 'unshift', 'splice'] as const).forEach(key => {
  const method = Array.prototype[key] as any
  arrayInstrumentations[key] = function(this: unknown[], ...args: unknown[]) {
    pauseTracking()
    const res = method.apply(this, args)
    enableTracking()
    return res
  }
})
```