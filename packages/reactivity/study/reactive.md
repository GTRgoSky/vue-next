### 声明全局 代理Map/只读Map
    const reactiveMap = new WeakMap<Target, any>()
    const readonlyMap = new WeakMap<Target, any>()

### reactive(target: object)
```js
    // entry:
    function reactive(target: object) {
        // if trying to observe a readonly proxy, return the readonly version.
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

    // mutableHandlers 是做什么得? - ProxyHandler 代理小助手
    // 请参考 mutableHandlers - mutableHandlers
    const mutableHandlers: ProxyHandler<object> = {
        get,
        set,
        deleteProperty,
        has,
        ownKeys
    }
```

### shallowReactive
```js
createReactiveObject(
    target,
    false,
    shallowReactiveHandlers,
    shallowCollectionHandlers
)
```

### readonly
```js
createReactiveObject(
    target,
    true,
    readonlyHandlers,
    readonlyCollectionHandlers
)
```

### shallowReadonly
```js
createReactiveObject(
    target,
    true,
    shallowReadonlyHandlers,
    readonlyCollectionHandlers
)
```

### createReactiveObject
```js

// 将类型 转为字符串用于识别
const objectToString = Object.prototype.toString
const toTypeString = (value: unknown): string => objectToString.call(value)

const toRawType = (value: unknown): string => {
    return toTypeString(value).slice(8, -1)
}

// 根据对象类型 分配 TargetType
function targetTypeMap(rawType: string) {
    switch (rawType) {
        case 'Object':
        case 'Array':
            return TargetType.COMMON
        case 'Map':
        case 'Set':
        case 'WeakMap':
        case 'WeakSet':
            return TargetType.COLLECTION
        default:
            return TargetType.INVALID
    }
}

// 根据 value 得属性 返回对应 TargetType 值
function getTargetType(value: Target) {
    return value[ReactiveFlags.SKIP] || !Object.isExtensible(value)
        ? TargetType.INVALID
        : targetTypeMap(toRawType(value))
}


// 所有劫持在这里创建
function createReactiveObject(
    target: Target,
    isReadonly: boolean,
    baseHandlers: ProxyHandler<any>, // [FX 做了什么]
    collectionHandlers: ProxyHandler<any> // [FX做了什么]
) {
    // 不是对象直接返回并警告
    if (!isObject(target)) {
        if (__DEV__) {
            console.warn(`value cannot be made reactive: ${String(target)}`)
        }
        return target
    }
    // target is already a Proxy, return it.
    // exception: calling readonly() on a reactive object
    // 如果是 原生对象,或者已经是一个 reactive 对象 直接返回 target
    if (
        target[ReactiveFlags.RAW] &&
        !(isReadonly && target[ReactiveFlags.IS_REACTIVE])
    ) {
        return target
    }
    // target already has corresponding Proxy
    // 赋值 代理 Map
    const proxyMap = isReadonly ? readonlyMap : reactiveMap
    // 如果已经有了代理直接返回
    const existingProxy = proxyMap.get(target)
    if (existingProxy) {
        return existingProxy
    }
    // only a whitelist of value types can be observed.
    // 只能观察值类型的白名单
    const targetType = getTargetType(target)
    // 如果是非 白名单,直接返回 对象
    if (targetType === TargetType.INVALID) {
        return target
    }

    // 创建代理, 根据 TargetType 类型 采用不同劫持
    const proxy = new Proxy(
        target,
        // 判断 targetType 是否是 Map/Set/WeakMap/WeakSet 
        targetType === TargetType.COLLECTION ? collectionHandlers : baseHandlers
    )

    // 代理地图 以 target 为key 存储 目标对象的 代理
    // 用途1： 拥有目标函数的缓存功能，节省开支
    proxyMap.set(target, proxy)
    return proxy
}
```

### isReactive
    1. 判断 value 若是只读, 如果是, 获取 value 得 __v_raw 对象(即原生对象)
    递归 isReactive
    2. 若不是 只读,直接判断 value 上是否含有 __v_isReactive

### isReadonly
    判断 value 上是否有 __v_isReadonly 标识

### isProxy
    判断 value 是否是被代理对象

### markRaw(value)
    创建一个 value 对象,不会被监听劫持