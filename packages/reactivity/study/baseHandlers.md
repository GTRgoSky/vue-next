## mutableHandlers
    当target是对象或者数组时, 对proxy进行 基础 配置
```js
    const mutableHandlers: ProxyHandler<object> = {
        get, // createGetter()
        set,
        deleteProperty,
        has,
        ownKeys
    }

```

## readonlyHandlers
    当目标函数时readonly时
    readonly的数据劫持。get触发readonlyGet ， set => true， deleteProperty => true
    只读不会触发 track 
```js
const readonlyGet = /*#__PURE__*/ createGetter(true)
const readonlyHandlers: ProxyHandler<object> = {
  get: readonlyGet,
  set(target, key) {
    if (__DEV__) {
      console.warn(
        `Set operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  },
  deleteProperty(target, key) {
    if (__DEV__) {
      console.warn(
        `Delete operation on key "${String(key)}" failed: target is readonly.`,
        target
      )
    }
    return true
  }
}
```

## shallowReactiveHandlers
    浅劫持
```js
// 当劫持设置为 浅 劫持时
// 只会对目标函数进行劫持，而不会递归操作他的子对象
const shallowGet = /*#__PURE__*/ createGetter(false, true)
// 不在关注 oldValue 是否是一个 Ref
const shallowSet = /*#__PURE__*/ createSetter(true)
const shallowReactiveHandlers: ProxyHandler<object> = extend(
  {},
  mutableHandlers,
  {
    get: shallowGet,
    set: shallowSet
  }
)
```

## shallowReadonlyHandlers

```js
const shallowReadonlyGet = /*#__PURE__*/ createGetter(true, true)
// 基于readonlyHandlers 重写 get
const shallowReadonlyHandlers: ProxyHandler<object> = extend(
  {},
  readonlyHandlers,
  {
    get: shallowReadonlyGet
  }
)
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

## createSetter
```js
function createSetter(shallow = false) {
  return function set(
    target: object,
    key: string | symbol,
    value: unknown,
    receiver: object
  ): boolean {
    const oldValue = (target as any)[key]
    if (!shallow) {
        // 获取value得原始对象
      value = toRaw(value)
      // 如果 target 不是数组 且原来是个 Ref 对象，则只替换 value 值
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
        : hasOwn(target, key) // key 是否在 target 的原型链上
    const result = Reflect.set(target, key, value, receiver)
    // don't trigger if target is something up in the prototype chain of original
    // 获取 receiver 的 原生对象 对比 target 如果相等， 触发 trigger 更新
    if (target === toRaw(receiver)) {
      if (!hadKey) {
        trigger(target, TriggerOpTypes.ADD, key, value)
      } else if (hasChanged(value, oldValue)) {
        trigger(target, TriggerOpTypes.SET, key, value, oldValue)
      }
    }
    return result
  }
}

```

## deleteProperty(target: object, key: string | symbol)
    删除对象属性，并触发一次 trigger

## has
    判断属性是否含有某 key ，并且 key不是 Symbol对象或者不是原生Symbol的属性值时，触发一次 track

## ownKeys
    触发一次 track，返回自身Key
