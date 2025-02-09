## 骨架部分
如何设计一个文件夹的层级呢？

结构如下

- assets存放静态文件
- components存放组件
	 - common存放公共组件
	 - custom特有组件
- constants存放常量或者环境变量等
- hooks存放一些特有hooks逻辑
- layout存放布局框架内容
	- context

## layout部分
最外层的layout一般使用内嵌的方式来进行书写，并将一些独立的逻辑抽离出来，我们可以只像需要的地方传入指定的组件即可。不需要处理额外的逻辑
**最外层**
我们可以称呼这个为baselayou组件，他内部只处理一些像是否折叠以及处理是否显示传入的子组件，比如：如果我们在外部来进行监听了用户点击折叠，可以通过参数来传递给baselayout组件，或者单纯的销毁传入的组件，这样都可以。
但是我们一般是通过插槽来将传入的子组件显示在对应的位置。如何知道有没有传入对应的插槽呢？
`defineSlots`可以获取到传入的slots是否存在指定的值
`const slots = defineSlots()`
`const showheader = computed(()=> Boolean(slots.header))`
这样我们就获取到了是否有具体的slots传入

## vue-router部分
具体抽象层级为：
-route
--index.ts
--guard //存放路由守卫
--basic //存放基本路由

vue-router需要和vue内置组件routerview一起搭配使用
步骤如下：
1：使用npm下载vue-router。此部分略过
2：创建router文件夹，并创建index.ts文件作为入口文件
3：
``` javascript
import type { RouteRecordRaw } from "vue-router"
//第一步引入vuerouter的类型
export async function setupRouter(app: App) {

  app.use(router);

  createRouterGuard(router);

  await router.isReady();

}

```

4：开始处理router对象
```js
export const router = createRouter({

  history: createWebHashHistory(),

  routes: baseRoutes as unknown as RouteRecordRaw[],

});
```

routes对象我们可以放入单独的路由对象，这里建议每个路由都单独放一个文件内以便处理
单个路由对象如下所示
```js
{
    path: "/redirect",
    component: Layouts,
    meta: {
      hidden: true
    },
    children: [
      {
        path: ":path(.*)",
        component: () => import("@/pages/redirect/index.vue")
      }
    ]
  },
```
5：处理路由守卫
```js
export function createRouterGuard(router: Router) {
  createProgressGuard(router);
  createRouteGuard(router);
}
```
建议单独封装
```js
function createPageGuard(router: Router) {

  const loadedPageMap = new Map<string, boolean>();

  router.beforeEach(async (to) => {

    // The page has already been loaded, it will be faster to open it again, you don’t need to do loading and other processing

    to.meta.loaded = !!loadedPageMap.get(to.path);

    // Notify routing changes

    setRouteChange(to);

    return true;

  });

  router.afterEach((to) => {

    loadedPageMap.set(to.path, true);

  });

}
```