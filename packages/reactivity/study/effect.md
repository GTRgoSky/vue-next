## targetMap 
    创建一个 全局 weakMap 对象，用于缓存所有 有 副作用的目标

## effectStack 
    副作用堆栈 一个数组List，先进后出 [FX 干嘛用的]

## activeEffect
    一个 正在被激活的 副作用 Function, 并且挂在了一些对象

## isEffect
    如果fn 存在 且具有副作用标示 _isEffect 返回true 表示具有副作用。

## effect(fn, options) - 创建一个副作用函数
    1. 如果 fn 已经是一个副作用函数 则 fn = fn.raw 
    (因为在createReactiveEffect中, 我们把 raw 赋值为 fn
    所以, 副作用函数的 raw 都是他的执行方法) 
    2. 用 createReactiveEffect 创建 真正的 副作用 effect 方法
    (其实这个 effect 就是 reactiveEffect 方法)
    3. 创建完成后 如果 options 没有配置 lazy 则立即执行一次 effect
    4. 返回 effect
```js

function effect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions = EMPTY_OBJ
): ReactiveEffect<T> {
  if (isEffect(fn)) {
    fn = fn.raw
  }
  const effect = createReactiveEffect(fn, options)
  if (!options.lazy) {
    effect()
  }
  return effect
}

```

## let uid = 0;
    创建一个标记

## createReactiveEffect(fn, option) - 真正执行创建
    1. 声明一个方法 effect = reactiveEffect (断言是一个副作用函数)
    2. 给 effect 绑定一些属性: [FX 属性作用未知]
        effect.id = uid++ // id
        effect._isEffect = true // 是否为 副作用函数
        effect.active = true // 是否激活
        effect.raw = fn // 真正副作用方法
        effect.deps = [] // Array<Dep> --- Dep = Set<ReactiveEffect> // 
            activeEffect.deps.push(dep)
            用于存储data对象的Set-Set中又存储的时DOM的副作用activeEffect
        effect.options = options // 配置参数
    3. reactiveEffect 方法作用:
        3.1 如果 effect.active 为 false [FX什么时候为false],
        return options.scheduler ?  undefined : fn() [FX options.scheduler 是什么]
        3.2 如果 effectStack 不包含 effect 进入判断
        3.3 执行 cleanup(effect) => 获取 deps 如果 deps 有值 删除deps中的所有副作用
        3.4 enableTracking() : trackStack.push(shouldTrack === true);
        3.5 effectStack.push(effect) 将 副作用 推入栈
        3.6 activeEffect = effect 副作用
        3.7 执行 fn() 返回值 - 这里如果 fn 执行后依然有副作用创建,则递归执行
        3.8 弹出尾部 effect
        3.9 resetTracking() :  
            const last = trackStack.pop() 
            shouldTrack = last === undefined ? true : last
            这个 shouldTrack 只有为 True 时,才会在 track 创建 追踪
        3.10 activeEffect = effectStack[effectStack.length - 1];

```js
type Dep = Set<ReactiveEffect>

interface ReactiveEffect<T = any> {
  (): T
  _isEffect: true
  id: number
  active: boolean
  raw: () => T
  deps: Array<Dep>
  options: ReactiveEffectOptions
}


interface ReactiveEffectOptions {
  lazy?: boolean
  scheduler?: (job: ReactiveEffect) => void
  onTrack?: (event: DebuggerEvent) => void
  onTrigger?: (event: DebuggerEvent) => void
  onStop?: () => void
  allowRecurse?: boolean
}

function createReactiveEffect<T = any>(
  fn: () => T,
  options: ReactiveEffectOptions
): ReactiveEffect<T> {
  const effect = function reactiveEffect(): unknown {
    if (!effect.active) {
      return options.scheduler ? undefined : fn()
    }
    if (!effectStack.includes(effect)) {
      cleanup(effect)
      try {
        enableTracking()
        effectStack.push(effect)
        activeEffect = effect
        return fn()
      } finally {
        effectStack.pop()
        resetTracking()
        activeEffect = effectStack[effectStack.length - 1]
      }
    }
  } as ReactiveEffect
  effect.id = uid++
  effect._isEffect = true
  effect.active = true
  effect.raw = fn
  effect.deps = []
  effect.options = options
  return effect
}

function cleanup(effect: ReactiveEffect) {
  const { deps } = effect
  if (deps.length) {
    for (let i = 0; i < deps.length; i++) {
      deps[i].delete(effect)
    }
    deps.length = 0
  }
}

let shouldTrack = true // 是否应该追踪
const trackStack: boolean[] = [] // 追踪栈

function enableTracking() {
  trackStack.push(shouldTrack)
  shouldTrack = true
}

function resetTracking() {
  const last = trackStack.pop()
  shouldTrack = last === undefined ? true : last
}

```

## pauseTracking/enableTracking/resetTracking

```js

    // 创建不被追踪的 (即非响应式的值) 记录
    // 官方解释: 避免数组的原生方法导致长度被追踪,这在某些情况下会导致无限循环
    export function pauseTracking() {
        trackStack.push(shouldTrack) // 上一个 对象追踪的状态
        shouldTrack = false // 这里才是当前 对象是否追踪的 状态
    }

    // 创建一个 响应式 值 记录
    export function enableTracking() {
        trackStack.push(shouldTrack)
        shouldTrack = true
    }

    // 弹出这个记录
    export function resetTracking() {
        const last = trackStack.pop()
        shouldTrack = last === undefined ? true : last
    }

```

## track(target, type, key) - 追踪器
> 以 target 和 key 为 key值 创建一个更新行为栈

    1. 如果 shouldTrack == false || activeEffect 不存在 直接返回不创建
    2. 在 targetMap 中 以 target 为 key 创建 depsMap
    3. 在 depsMap 中 以 key 为 key值 创建 dep: Set
    4. 判断dep 是否含有 当前激活的 activeEffect 没有下一步:
    5. dep.add(activeEffect) -- activeEffect.deps.push(dep); 一个相互存储
        dep以Set形式存储属于当前key值得所有DOM副作用
        activeEffect.deps以数组形式 存储所有 该副作用影响到得keyMap

```js
function track(target: object, type: TrackOpTypes, key: unknown) {
  if (!shouldTrack || activeEffect === undefined) {
    return
  }
  let depsMap = targetMap.get(target)
  if (!depsMap) {
    targetMap.set(target, (depsMap = new Map()))
  }
  let dep = depsMap.get(key)
  if (!dep) {
    depsMap.set(key, (dep = new Set()))
  }
  if (!dep.has(activeEffect)) {
    dep.add(activeEffect)
    activeEffect.deps.push(dep)
  }
}
```

## trigger
    1. 从 targetMap 以 target 获取对应 depsMap
    2. 没有表面没有副作用直接返回,有进行下一步:
    3. 声明一个 effects: Set
    4. 声明一个 add(effectsToAdd: Set<ReactiveEffect> | undefined) 函数
        这个函数用于向 effects对象: Set 中添加 副作用
    5. type得区别:
        5.1 如果 type 是 clear 则 意味着触发所有绑定在该 target 得副作用
        5.2 如果 key是一个 length 并且 target 是一个 Array<Set>.则:
            遍历 depsMap(因为代表着它也是一个数组Map,key都是数组下标)
            if (key === 'length' || key >= (newValue as number)) 通过这个判断执行 add(dep)
        5.3 进入else 如果 key 不是 undefiend 则直接 add(depsMap.get(key))
        5.4 如果 type 分别是 ADD/DELETE/SET等执行一些逻辑,将响应副作用加入 effects
    6. 声明 run 函数,执行 effect() --- effects.forEach(run)


```js
// 触发器
export function trigger(
  target: object,
  type: TriggerOpTypes,
  key?: unknown,
  newValue?: unknown,
  oldValue?: unknown,
  oldTarget?: Map<unknown, unknown> | Set<unknown>
) {
  const depsMap = targetMap.get(target)
  if (!depsMap) {
    // never been tracked
    return
  }

  const effects = new Set<ReactiveEffect>()
  const add = (effectsToAdd: Set<ReactiveEffect> | undefined) => {
    if (effectsToAdd) {
      effectsToAdd.forEach(effect => {
        if (effect !== activeEffect || effect.options.allowRecurse) {
          effects.add(effect)
        }
      })
    }
  }

  if (type === TriggerOpTypes.CLEAR) {
    // collection being cleared
    // trigger all effects for target
    depsMap.forEach(add)
  } else if (key === 'length' && isArray(target)) {
    depsMap.forEach((dep, key) => {
      if (key === 'length' || key >= (newValue as number)) {
        add(dep)
      }
    })
  } else {
    // schedule runs for SET | ADD | DELETE
    if (key !== void 0) {
      add(depsMap.get(key))
    }

    // also run for iteration key on ADD | DELETE | Map.SET
    switch (type) {
      case TriggerOpTypes.ADD:
        if (!isArray(target)) {
          add(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            add(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        } else if (isIntegerKey(key)) {
          // new index added to array -> length changes
          add(depsMap.get('length'))
        }
        break
      case TriggerOpTypes.DELETE:
        if (!isArray(target)) {
          add(depsMap.get(ITERATE_KEY))
          if (isMap(target)) {
            add(depsMap.get(MAP_KEY_ITERATE_KEY))
          }
        }
        break
      case TriggerOpTypes.SET:
        if (isMap(target)) {
          add(depsMap.get(ITERATE_KEY))
        }
        break
    }
  }

  const run = (effect: ReactiveEffect) => {
    if (__DEV__ && effect.options.onTrigger) {
      effect.options.onTrigger({
        effect,
        target,
        key,
        type,
        newValue,
        oldValue,
        oldTarget
      })
    }
    if (effect.options.scheduler) {
      effect.options.scheduler(effect)
    } else {
      effect()
    }
  }

  effects.forEach(run)
}
```

## 流程:
    1. effect 声明副作用 - 是用来影响页面DOM的
    2. 立即执行 effect
    3. cleanup 清楚缓存副作用
    4. enableTracking 加入追踪
    5. effectStack.push(effect)
    6. 激活当前activeEffect
    7. 执行 真正 fn() - 这里其实是 runtime-core的 componentEffect (实例update时)
        7.1 执行 setup 创建里面得 响应式数据
    8. track(target, type, key) 创建追踪器 这个target就是被追踪的对象 - 追踪数据的
    9. targetMap.set(target, (depsMap = new Map()))
    10. depsMap.set(key, (dep = new Set()))
    11. dep.add(activeEffect) 将副作用加入 dep
    12. activeEffect.deps.push(dep) 将data的DOM副作用存入DOM的deps
    13. effectStack.pop(); 弹出最后一个
    14. resetTracking() 删除当前追踪状态
    15. activeEffect = effectStack[effectStack.length - 1] 为将激活的副作用赋值
    16. 下一次 track