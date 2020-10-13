## Ref
> packages\reactivity\src\ref.ts
```js
const r = ref(rawValue, [shallow = false])
```

### ref()

    1. 执行 createRef(rawValue)
    2. 判断 rawValue 是否是一个Ref对象，如果是直接返回，如果不是则实例化 RefImpl 
        2.1 RefImpl 接受 rawValue, shallow 两个参数，设置
            this._rawValue = rawValue;
            this.__v_isRef = true;
            this._shallow = _shallow;
            this._value = convert(rawValue) // 转换; 
    4. 执行 convert(_rawValue)
        4.1 convert 判断 _rawValue 是否是 Object 如果不是返回原值，是则 将该对象 用reactive创建
    5. 获取 value 时 (get)
        情景一： 未绑定副作用,但获取value
            5.1 执行toRaw(this:当前对象)  toRaw -- 返回由 reactive 或 readonly 方法转换成响应式代理的普通对象。这是一个还原方法，可用于临时读取，访问不会被代理/跟踪，写入时也不会触发更改。
            5.2 执行 track【轨迹】(目标对象，类型，key) 返回类型值
        情景二：有副作用
            5.1 全局有一个 targetMap 对象 。以 target 为 key 存储一个 depsMap 的 Map对象
            5.2 depsMap 。以 target 上的 key 为 key 创建一个 dep 的 Map 对象
            5.3 dep 存储 副作用  dep.add(activeEffect)
            5.4 全局 activeEffect = reactiveEffect 副作用 activeEffect.deps.push(dep)
            5.5 这里存储的 targetMap 将在 trigger 获取使用
        情景三：
            只创建了值，但是没有执行get/set。不会触发 track。
            depsMap 因为没有执行 track 则不会保存轨迹，所以set直接被return;
    6. 设置 value 时 接受一个 newVal (set)
        6.1 先判断 若之前声明 ref 时通过 ref(obj)形式 则不再进入 ref 的set，如果之前是 ref(基本类型)则进入判断。
        6.2 判断新老值是否有变更。若有进入判断
        6.3 this._rawValue = newVal
        6.4 this._value = convert(newVal)
        6.5 执行 trigger 触发更新

### shallowRef()

    1. 执行createRef(rawValue, true), 返回一个不被 响应式代理 的双向绑定（但是也加入到了副作用数组中）。但是强制更新或者其他响应式更新时，会更新它
    原因是 this._value = _shallow === true ? _rawValue : convert(_rawValue);
    所以在用 ref.value.xxx = xxx 时，不会进入 ref 的set逻辑。
    因为劫持的是 value。
    2. 如果 rawValue 是一个 基础类型则不受约束
    3. 如果直接修改 ref.value = {} 则不受此约束

### isRef
    
    检查一个值是否为一个 ref 对象。 ref 特有标识符 __v_isRef === true

### unref(ref)

    如果参数是一个 ref 则返回它的 value，否则返回参数本身。它是 val = isRef(val) ? val.value : val 的语法糖。

### proxyRefs(objectWithRefs)

    判断 objectWithRefs 是否是一个 Reactive，如果是直接返回。
    若不是，返回一个 objectWithRefs 的代理：
    get: 返回一个 unref(target.key)
    set: 若 oldValue 是一个 ref 且 newValue 不是一个 ref 则 oldValue.value = newValue
    若不满足上述条件 target.key = newValue

### customRef(factory = (track, trigger) => {get:Function,set:Function})
> 用于自定义一个 ref，可以显式地控制依赖追踪和触发响应，接受一个工厂函数，两个参数分别是用于追踪的 track 与用于触发响应的 trigger，并返回一个带有 get 和 set 属性的对象
```js
customRef((track, trigger) => {
    return {
      get() {
        track()
        return value
      },
      set(newValue) {
        clearTimeout(timeout)
        timeout = setTimeout(() => {
          value = newValue
          trigger()
        }, delay)
      },
    }
  })
```

### toRef
> 可以用来为一个 reactive 对象的属性创建一个 ref。这个 ref 可以被传递并且能够保持响应性。

    1. 如果是 Ref 直接返回该 ref
    2. 否则进入 ObjectRefImpl 
    3. 创建 __v_isRef = true，接受 object 和 key
    4. 获取直接返回 this._object[this._key]， 赋值直接 this._object[this._key] = newVal
    5. 因为 object 是一个 reactive 对象，所以此功能只是给返回对象 打了一个 __v_isRef 标签。
    其余逻辑走的还是 reactive 对象的逻辑

### toRefs 
> 把一个响应式对象转换成普通对象，该普通对象的每个 property 都是一个 ref ，和响应式对象 property 一一对应。

    实际是便利传入对象的可枚举属性，然后用 toRef 进行处理