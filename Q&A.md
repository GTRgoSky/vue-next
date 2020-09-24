## Q: createApp(__script) 都做了什么
## A:
0. 接受一个 __script 对象(由编译工具产生),拥有 render 函数和一些其他编译产生的对象. 第二个参数接受根 props
1. 生产了一个 const context = createAppContext(); 返回一个对象,拥有 app 应用程序的 公共对象
(eg: app.config.globalProperties)
2. 生成一个 installedPlugins 的Set实例 用于存储 app.use 的插件
3. 创建一个 isMounted 标记, 用于判断当前 组件状态 是否被初始化
4. 创建一个 app 应用程序,将上述 context上下文的app属性指向这个 app .并将app的_context指向context;app 绑定一些属性:(绑定如下,只写主要:)
    4.1. 组件的_script对象
    4.2 _uid
    4.3 config获取 context.config
    4.4 app 的use ,mixin ,component ,directive, mount ,unmount ,provide 这些全局方法 
5. 返回app应用程序

## Q: app.mount 做了什么
## A: 
1. 创建当前组件Vnode
2. 执行 render 函数
3. 将 isMounted 标记为 true
4. 返回 vnode 的 component 的 proxy 代理

## app.config.globalProperties 为什么不能在setUp中引用

## Instance 创建并暴露的逻辑、

## 如何兼容Vue2.0