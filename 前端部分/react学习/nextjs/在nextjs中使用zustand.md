---
title: 在 Next.js 中使用 Zustand
aliases:
  - 在 Next.js 中使用 Zustand
tags:
  - Next.js
  - React
  - Zustand
  - 水合问题
status: 待完善
created: 2026-06-21
---

# 在 Next.js 中使用 Zustand 🧩

## 问题背景 📌

## 水合问题 💧

在 Next.js 中有一个很严重的问题就是水合问题。何为水合问题？要了解水合问题这个概念之前，我们重新回到 Next.js 的基本概念：服务端渲染。

前端和 Web 的道路已经走了数十年。在很多前端框架横空出世以后，前后端分离、前端工程化的概念冲击全世界。每个网站都可以实现以前无法想象的效果，公司机构以及网站把越来越多的逻辑和操作步骤放在了前端进行，但这也带来了一个问题：前端项目的体积越来越大，网页的加载速度越来越慢。

为了弥补这个问题，服务端渲染这个概念被提了出来。何为服务端渲染？就是在服务器上先把 React 组件渲染成浏览器可以直接读取的 HTML，这样可以让用户更快看到首屏内容；但客户端交互通常仍然依赖后续下载 `JS bundle` 并完成 hydration。

但是，服务端渲染有一个严重的问题，就是渲染是在服务器上进行的，服务器上自然是没有浏览器 API 的。现代前端框架的很多功能都依赖于浏览器 API，而且还带来了水合问题。

水合问题简而言之，就是服务端生成的 HTML 和客户端第一次渲染结果不一致。这通常发生在我们在服务端进行了某些奇怪的操作时，比如在服务端调用了某些随机数生成、时间戳或者使用了和客户端初始状态不一致的 store。

Next.js 等框架的渲染步骤：

1. 服务端渲染 React 组件。
   -> 生成 HTML 字符串。
2. 浏览器收到 HTML。
   -> 先直接显示出来，用户能看到页面。
3. 浏览器加载 JS bundle。
   -> React 在客户端重新跑一遍组件。
   -> 尝试把事件、状态、组件树绑定到已有 HTML 上。
4. 如果客户端第一次渲染结果和已有 HTML 对不上。
   -> hydration mismatch / 水合问题。

## Zustand 的问题 ⚠️

如果我们在 Next.js 里用全局单例 store，或者服务端初始状态和客户端初始状态不一致，就很容易出现水合问题，甚至可能出现不同请求之间状态串用。为了解决这个问题，我们需要使用一些更谨慎的步骤：

- Zustand 导出 Provider 和 store 创建函数。
- 创建一个中间组件，在该组件内接收 props，并使用刚才拿到的 Provider 和 store 创建一个真实的 Provider，传递给它的子组件。
- 在服务端拿到一些初始化的值以后，向该中间组件内注入。
- Provider 组件内初始化 store。
- 使用 Hooks 访问 store。

接下来是代码演示，分为两种：纯原生写法和使用 Zustand 官方插件的写法。

## 原理 🧠

通过 Zustand 的 vanilla store 工厂函数，把 store 实例的创建放到客户端 Provider 渲染时。服务端组件先获取数据，再通过 props 传给客户端 Provider。客户端 Provider 用这些初始数据创建 store，并通过 React Context 把这个 store 实例提供给下面的客户端组件。最后再封装一个 hook，用 `useContext` 拿到 store，并用 Zustand 的 `useStore` 读取状态。

Provider 组件里一般要用 `useRef` 保存 store 实例，避免组件重新渲染时重复创建 store，不然状态可能会被重新初始化。

## 原生写法 🛠️

- 使用 `useRef` 创建一个变量，`useRef` 创建的变量将不会跟随组件的重新渲染而清空。
- 创建一个 `createStore` 的工厂函数，再用 `useRef` 持有 store 实例，避免组件重新渲染时重复创建 store。
- 创建 Context。
- 创建一个组件，使用 Context 和 store，并准备接收外部的 props 来初始化 store。
- 创建一个 hook，供其他组件使用。
- 在服务端组件调用 Provider，传入参数。
- 在 Provider 内初始化 store。
- 其他页面使用 hook 来消费 store。

```tsx
'use client'; // 因为涉及 Context 和 Hooks，必须是 Client Component

import { createContext, useContext, useRef } from 'react';
import { createStore, useStore } from 'zustand';

// 1. 制造“模具” (Store Factory)：注意这里用的是 createStore 而不是 create
const createUserStore = (initialProps?: Partial<UserState>) => {
  return createStore<UserStoreType>()((set, get) => ({
    ...initialUserState, // 数据
    ...initialProps, // 把服务端传来的初始数据覆盖上去
    ...actions, // 动作
  }));
};

// 2. 创建 React Context 实例，专门存放上面模具造出来的 store
export const UserStoreContext = createContext<ReturnType<typeof createUserStore> | null>(null);

// 3. 封装出暴露给外部的 Provider 组件，负责在请求进来的那一刻真正 new 出实例
export function UserProvider({
  children,
  initialData,
}: {
  children: React.ReactNode;
  initialData?: Partial<UserState>;
}) {
  const storeRef = useRef<ReturnType<typeof createUserStore>>();

  if (!storeRef.current) {
    storeRef.current = createUserStore(initialData); // 仅初始化一次
  }

  return (
    <UserStoreContext.Provider value={storeRef.current}>
      {children}
    </UserStoreContext.Provider>
  );
}

// 4. 封装成业务层熟悉的 useUserStore hook
export function useUserStore<T>(selector: (state: UserStoreType) => T): T {
  const store = useContext(UserStoreContext);
  if (!store) throw new Error('必须在 UserProvider 内部使用 useUserStore!');
  return useStore(store, selector);
}


```

注入：

```tsx
// src/app/layout.tsx
import { UserProvider } from '@/stores/user';

export default function RootLayout({ children }) {
  // 服务端拿到初始数据
  const initialData = { status: 'authenticated', user: { name: 'Motoko' } };

  return (
    <UserProvider initialData={initialData}>
      {children}
    </UserProvider>
  );
}
```

消费：

```tsx
// src/features/auth/loginForm.tsx
import { useUserStore } from '@/stores/user';

export function LoginForm() {
  // 直接使用 hook 取值
  const status = useUserStore((state) => state.status);
  const login = useUserStore((state) => state.login);
}
```

## 使用第三方插件 🔌

这类第三方封装通常会帮我们省略 Context、Provider 和 `useStore` 的样板代码，其他步骤也都是一样的。

- 创建一个 store。
- 使用插件接收 store 并导出 Provider 和 hook。
- 在服务端页面使用 Provider 并注入需要使用的数据。
- 其他页面使用 hook 进行消费。

创建：

```tsx
// src/stores/user/index.ts
'use client';

import { createStore } from 'zustand';
import { createContext } from 'zustand-utils';

import { initialUserState, UserState } from './initialState';

export interface UserStoreType extends UserState {
  login: (email: string, password?: string) => Promise<void>;
  logout: () => Promise<void>;
  fetchSession: () => Promise<void>;
}

// 1. 制造“模具” (Store Factory) —— 与原生写法的第 1 步完全一致
export const createUserStore = (initialProps?: Partial<UserState>) => {
  return createStore<UserStoreType>()((set, get) => ({
    ...initialUserState,
    ...initialProps,

    // ... 这里是 login、logout 等业务代码，为了排版
  }));
};

// 2. 使用插件黑盒一键生成 Provider 和 useStore
export const {
  Provider,
  useStore: useUserStore,
} = createContext<ReturnType<typeof createUserStore>>();

// 3. 创建好一个 Provider 组件，进行一次客户端级别的初始化
export function UserProvider({
  children,
  initialData,
}: {
  children: React.ReactNode;
  initialData?: Partial<UserState>;
}) {
  return (
    <Provider createStore={() => createUserStore(initialData)}>
      {children}
    </Provider>
  );
}
```

注入：

```tsx
// src/app/layout.tsx
import { UserProvider } from '@/stores/user';

const mockSessionUser = { name: 'Motoko' };

export default function RootLayout({ children }) {
  // 在这里注入服务端拿到的初始数据
  return (
    <UserProvider initialData={{ user: mockSessionUser, status: 'authenticated' }}>
      {children}
    </UserProvider>
  );
}
```

消费：

```tsx
import { useUserStore } from '@/stores/user';

export function LoginForm() {
  // 直接使用 hook 取值
  const status = useUserStore((state) => state.status);
  const login = useUserStore((state) => state.login);
}
```

## 待创建术语笔记 📝

- [ ] [[Hydration Mismatch]]：解释服务端渲染结果和客户端首次渲染结果不一致时为什么会报错。 #待创建术语笔记
- [ ] [[Zustand Vanilla Store]]：整理 `createStore`、Provider、Context 和 `useStore` 的组合方式。 #待创建术语笔记
- [ ] [[Next.js Client Component]]：补充为什么涉及浏览器状态和 Hooks 的组件需要声明 `'use client'`。 #待创建术语笔记
