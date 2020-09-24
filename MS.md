## Q: 为什么使用proxy 而不再用 defineProperty
## A: 
1. 由于 Object.defineProperty 只能对属性进行劫持，需要遍历对象的每个属性，如果属性值也是对象，则需要深度遍历。<br>
而 Proxy 直接代理对象，不需要遍历操作。<br>
2. 由于 Object.defineProperty 劫持的是对象的属性，所以新增属性时，需要重新遍历对象，对其新增属性再使用 Object.defineProperty 进行劫持。
也正是因为这个原因，使用vue给 data 中的数组或对象新增属性时，需要使用 vm.$set 才能保证新增的属性也是响应式的。<br>
如果采用 proxy 实现，Proxy 通过 set(target, propKey, value, receiver) 拦截对象属性的设置，是可以拦截到对象的新增属性的<br>
也正是因为这个特性,所以Vue3可以对数组进行劫持了(例入下标)<br>
3. Proxy支持13种拦截操作，这是defineProperty所不具有的
4. 新标准性能红利
Proxy 作为新标准，长远来看，JS引擎会继续优化 Proxy，但 getter 和 setter 基本不会再有针对性优化。
5. Proxy兼容性差

## Q: 为什么不再用new Vue 而是 return app
## A: 
1. 解决Vue.mixin/component/directive/use/property.xxx等全局注册 影响全局Vue导致逻辑混乱,数据相互影响.