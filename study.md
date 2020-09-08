#### 创建APP (createApp)
packages\runtime-dom\src\index.ts -> createApp
packages\runtime-core\src\renderer.ts -> 声明createRenderer -> 声明/调用 baseCreateRenderer
packages\runtime-core\src\apiCreateApp.ts -> createAppAPI -> createApp

#### 返回应用实例 (createAppContext)
packages\runtime-core\src\apiCreateApp.ts -> createAppContext

#### 劫持在那里创建：
packages\reactivity\src\reactive.ts => createReactiveObject方法

#### 生命周期/钩子函数
packages\runtime-core\src\apiLifecycle.ts => injectHook方法

#### 模板编译
packages\runtime-core\src\componentProxy.ts -> PublicInstanceProxyHandlers 方法

#### watchEffect逻辑
packages\runtime-core\src\apiWatch.ts -> watch方法（118）