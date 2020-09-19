## 文件目录
https://juejin.im/post/6844903957421096967
## 案例在TestLoL的Vite尝试

## 创建APP (createApp)
    1. import App from './App.vue'; - 生成一个App.vue 的 render 函数，并绑定在 export 导出的 render 对象上
```javascript
import HelloWorld from '/src/components/HelloWorld.vue'

const __script = {
    name: 'App',
    components: {
        HelloWorld
    }
}

import {render as __render} from "/src/App.vue?type=template" // 发送请求，得到App.vue的render函数

__script.render = __render
__script.__hmrId = "/src/App.vue"
__script.__file = "/开发相关/codeProject/TestLOL/vite/src/App.vue"
export default __script


// /src/App.vue?type=template 
import {createVNode as _createVNode, resolveComponent as _resolveComponent, Fragment as _Fragment, openBlock as _openBlock, createBlock as _createBlock} from "/@modules/vue.js"

const _hoisted_1 = /*#__PURE__*/
_createVNode("img", {
    alt: "Vue logo",
    src: "/src/assets/logo.png"
}, null, -1 /* HOISTED */
)

export function render(_ctx, _cache, $props, $setup, $data, $options) {
    const _component_HelloWorld = _resolveComponent("HelloWorld")

    return (_openBlock(),
    _createBlock(_Fragment, null, [_hoisted_1, _createVNode(_component_HelloWorld, {
        msg: "Hello Vue 3.0 + Vite"
    })], 64 /* STABLE_FRAGMENT */
    ))
}
```
    2. createApp(__script) => ensureRenderer().createApp(...args)
    packages\runtime-dom\src\index.ts -> createApp 初始化，接受一个祖先 Render 函数对象
        2.1 调用 ensureRenderer 方法，返回一个renderer函数
        2.2 renderer 函数 =  packages\runtime-core\src\renderer.ts -> createRenderer 接受一个 rendererOptions ，rendererOptions参数是为了解释runtime-core的编译器（packages/runtime-dom/src/nodeOps.ts）【对应行为】
        2.3 createRenderer 返回 baseCreateRenderer 函数 接受参数为上一步的 rendererOptions，并在内部对上述定义行为映射 core 中的对应方法 【427】
        2.4 并最终返回一个 createApp = createAppAPI(render, hydrate) 接受 baseCreateRenderer 生成的render = 返一个 可以返回 Vue 对象 的方法
---------
    3. createApp(__script).mount('#app'); 
    在 packages/runtime-dom/src/index.ts 的 createApp  执行 const app = ensureRenderer().createApp(...args) 触发了上述方法 
    获取到 app 对象，并绑定一个mount方法，并在 调用 mount 时， 调用原来 createAppAPI（APP【packages/runtime-core/src/apiCreateApp.ts】） 上挂载的 mount 函数（重载该方法时会先保存这个函数）
```javascript
export const createApp = ((...args) => {
  const app = ensureRenderer().createApp(...args)

  if (__DEV__) {
    injectNativeTagCheck(app)
  }

  const { mount } = app // 缓存 createAppAPI 方法的 mount
  // 赋值 mount 挂载方法
  // createApp(App).mount('#app')
  app.mount = (containerOrSelector: Element | string): any => {
    const container = normalizeContainer(containerOrSelector)
    if (!container) return
    const component = app._component
    if (!isFunction(component) && !component.render && !component.template) {
      component.template = container.innerHTML
    }
    // clear content before mounting
    container.innerHTML = ''
    // packages/runtime-core/src/apiCreateApp.ts mount【218】
    const proxy = mount(container)
    container.removeAttribute('v-cloak')
    container.setAttribute('data-v-app', '')
    return proxy
  }

  return app
}) as CreateAppFunction<Element>
```
        3.1 执行 packages/runtime-core/src/apiCreateApp.ts 的 mount 为 render 对象创一个 Vnode， vnode的 type 指向 render 对象 
            再执行 baseCreateRenderer 的 render 方法
        3.2 调用 baseCreateRenderer 自身 patch 执行比对 - 若是组件见 4 
    4. patch：自身 = baseCreateRenderer
        4.1 执行自身 processComponent 加工组件 根据当前状态选择组件生命周期 执行对应周期函数 
            4.1.1 processComponent 有2种情况 一种初始化如 4.2 另一种 更新 执行 updateComponent -执行 4.5的 update 函数
        4.2 - mountComponent 初始化组件 
        4.3 - createComponentInstance 创建组件实利
        4.4 - setupComponent （setUp）周期 
        4.5 - setupRenderEffect (绑定组件实例 update方法 方法内容：)  - update 被 updateComponent 调用
            4.5.1 - renderComponentRoot (__script.render 函数执行的位置) 
            4.5.2 - 执行render 
            4.5.3 - openBlock 
            4.5.4 - createBlock 
            4.5.5 - normalizeChildren （每次创建Vnode都会执行， 暂时理解为修整 子节点的flag 和 对不同子节点状态的处理）（最终都收集在VNode） 
            4.5.6 - closeBlock
            4.5.7 - normalizeVNode 根据 render 函数 返回的 Vnode 判断 类型， 做相应返回
    5.  返回一个 vnode.component.proxy 代理 - 返回 proxy
        
监听 绑定 未读
------
setupRenderEffect 
## 返回应用实例 (createAppContext)
packages\runtime-core\src\apiCreateApp.ts -> createAppContext

## 劫持在那里创建：
packages\reactivity\src\reactive.ts => createReactiveObject方法

## 生命周期/钩子函数
packages\runtime-core\src\apiLifecycle.ts => injectHook方法

## 模板编译
packages\runtime-core\src\componentProxy.ts -> PublicInstanceProxyHandlers 方法

## watchEffect逻辑
packages\runtime-core\src\apiWatch.ts -> watch方法（118）