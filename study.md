## 文件目录
https://juejin.im/post/6844903957421096967
## 案例在TestLoL的Vite尝试

## 创建APP (createApp)
    1. import App from './App.vue'; - 生成一个App.vue 的 render 函数，并绑定在 export 导出的 render 对象上
```javascript
// 代码块 1
<template>
	<h1>666</h1>
	<h1>{{ msg }}</h1>
	<div>
		<HelloWorld msg="Hello Vue 3.0 + Vite" />
	</div>
	<h1>777</h1>
</template>

<script>
import HelloWorld from './components/HelloWorld.vue';

export default {
	name: 'App',
	data() {
		return {
			msg: 'Hello World',
		};
	},
	components: {
		HelloWorld,
	},
};
</script>

import HelloWorld from '/src/components/HelloWorld.vue';

const __script = {
    name: 'App',
    data() {
        return {
            msg: 'Hello World',
        };
    },
    components: {
        HelloWorld,
    },
};

import {render as __render} from "/src/App.vue?type=template"
__script.render = __render
__script.__hmrId = "/src/App.vue"
__script.__file = "/开发相关/codeProject/TestLOL/vite/src/App.vue"
export default __script


// 代码块 2
// /src/App.vue?type=template 
import {createVNode as _createVNode, toDisplayString as _toDisplayString, resolveComponent as _resolveComponent, Fragment as _Fragment, openBlock as _openBlock, createBlock as _createBlock} from "/@modules/vue.js"

const _hoisted_1 = /*#__PURE__*/ 可以看到这里生成了一个Vnode 并且 赋值给了最终 render 函数的子节点上，并打上了 1 的 flagTag，说明他是一个静态节点
_createVNode("h1", null, "666", -1 /* HOISTED */
)
const _hoisted_2 = /*#__PURE__*/
_createVNode("h1", null, "777", -1 /* HOISTED */
)

// 这个 render 挂在 在 __script的 __render 属性上
export function render(_ctx, _cache, $props, $setup, $data, $options) {
    const _component_HelloWorld = _resolveComponent("HelloWorld")

    return (_openBlock(),
    _createBlock(_Fragment, null, [_hoisted_1, _createVNode("h1", null, _toDisplayString($data.msg), 1 /* TEXT */
    ), _createVNode("div", null, [_createVNode(_component_HelloWorld, {
        msg: "Hello Vue 3.0 + Vite"
    })]), _hoisted_2], 64 /* STABLE_FRAGMENT */
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
        3.1 执行 packages/runtime-core/src/apiCreateApp.ts 的 mount 为 render 对象创一个 Vnode， 
            vnode的 type 指向 render 对象 也就是 template文件返回的 __script 对象 （见步骤1代码块）
            再执行 baseCreateRenderer 的 render 方法
        3.2 调用 baseCreateRenderer 自身 patch 执行比对 - 若是组件见 4 
    4. patch：自身 = baseCreateRenderer
        4.1 执行自身 processComponent 加工组件 根据当前状态选择组件生命周期 执行对应周期函数 
            4.1.1 processComponent 有2种情况 一种初始化如 4.2 另一种 更新 执行 updateComponent -执行 4.5的 update 函数
        4.2 - mountComponent 初始化组件 
        4.3 - createComponentInstance 创建组件实利
        4.4 - setupComponent （setUp）周期 - setupStatefulComponent
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
## 如何将子元素插入到Dom中
    1. 先通过一个Demo看到插入顺序：
```javascript
    <template>
        <h1>666</h1>
        <h1>{{ msg }}</h1>
        <div>
            <HelloWorld msg="Hello Vue 3.0 + Vite" /> => (内容)
            <h1>{{ msg }}</h1> 
            <button @click="count++">count is: {{ count }}</button>
            <p>Edit <code>components/HelloWorld.vue</code> to test hot module replacement.</p>
        </div>
        <h1>777</h1>
    </template>

    // 他们的insert顺序是：
    // child -- parent
    <h1>666</h1> -- <div id="app" data-v-app></div> 
    <h1>Hello World</h1> -- <div id="app" data-v-app></div> 
    <h1>Hello Vue 3.0 + Vite</h1> -- <div></div>
    <button>count is: 0</button> -- <div></div>
    Edit -- <p></p>
    <code>components/HelloWorld.vue</code> -- <p></p>
    to test hot module replacement. -- <p></p>
    <p></p> -- <div></div>
    <div></div> -- div id="app" data-v-app></div> 
    <h1>777</h1> -- <div id="app" data-v-app></div> 
```

    2. 在请求到template（代码块2） 文件后，会先将静态节点 执行 _createVNode， 并保存他们的Vnode
        最终生成一个 render 函数，保存在 __script的 __render 属性上，这个render 最终会生成真正的dom。
        Vue 3 这边把最终的Dom都生成在了 Fragment dom下，所以现在不用 在 模板上 用dev 包裹 一层

    3 先从子到父 的把静态 vnode 执行完毕 生成静态 Vnode 对象（这个对象被保存在了每个组件最终生成的render函数中）。【依次 执行 _createVNode】
        3.1 执行 Vnode p -> 生成 Vnode文本节点 Edit -> 生成 Vnode element元素 code -> Vnode文本节点 -> 重写 Vnode p的type(因为它含有多个子 vnode) 和 children 返回 Vnode p；
        3.2 进入到父组件 生成静态标签 Vnode h1 666 和 h1 777；

    4 开始执行 mount(#app) [rootComponent -> 就是 根组件 在这里是 App.vue]
        4.1 创建 App 的 Vnode - 执行 render - 执行 patch【packages/runtime-core/src/renderer.ts】
        mount(#app)的 patch过程：
        4.2 processComponent - mountComponent - createComponentInstance 【FX 创建实例 过程】- 
            4.2.1 给 instance 绑定了 uid
            4.2.2 ctx 映射到 instance 自己
            4.2.3 绑定了 root 指向 根节点的 instance 【跟节点的 instance 指向 自己】
            4.2.4 绑定了 emit 【FX】
            4.2.5 这个 实例 应该就是 vue3中 暴露给我们的 ctx（setup中第二个参数）
        4.3 setupComponent - setupStatefulComponent - finishComponentSetup 【FX】
            4.3.1 setupComponent 先初始化props和slots【FX 过程】 然后 判断 当前 是否是 组件，如果是 执行 setupStatefulComponent 否则返回 undefined。
            4.3.2 setupStatefulComponent 这里主要是 判断 是否是 Composition API 。若是 【FX setup逻辑过程】 先忽略。若不是 执行 finishComponentSetup
            4.3.3 finishComponentSetup 完成setUp初始化， 将__script.render 函数赋值给 组件实例 instance， 并且在这里做了Vue2.x 的兼容 - 结束
        4.4 执行 setupRenderEffect 【FX 创建副作用过程】创建 update，这里执行了一个effect 方法， 应该是副作用队列
        patch 过程 结束。
    5. 开始执行 __script 的 render 函数 子节点的 【FX 谁执行的这个函数】
    6. 创建 Vnode - 依次生成 最终生成 Fragment 根 Vnode
        6.1 执行 processFragment 【FX 过程】
    7. 依次执行 patch【这个patch 谁 执行的 FX】 通过 processElement -> mountElement -> hostSetElementText -> hostInsert 将子元素真正插入 dom 
        7.1 hostCreateElement 映射 createElement 创建 标签 【packages/runtime-dom/src/nodeOps.ts】
        7.2 hostSetElementText 映射 setElementText 将文本内容插入标签
        7.3 hostInsert 映射 insert 将 dom 使用 insertBefore 插入对应位置

    -- 整个流程还没有走完 [patch的整套逻辑]

------
 
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

## Block VNode 创建过程:
https://segmentfault.com/a/1190000024569143
「Vue3」针对靶向更新而提出的概念，它的本质是动态节点对应的 VNode