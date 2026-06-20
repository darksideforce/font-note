---
title: Next.js 的中间件：路由校验
aliases:
  - Next.js 的中间件：路由校验
  - Next.js Middleware 路由校验
tags:
  - Next.js
  - Middleware
  - 路由校验
  - 鉴权
status: 待完善
created: 2026-06-21
---

# Next.js 的中间件：路由校验 🛡️

提到 Next.js 的服务端渲染，还有一个绕不开的东西就是中间件。我先来简单说一下中间件在一个什么位置。

当浏览器发起请求时，这个请求大致会先后经过：

```text
浏览器 (Client) ---> 中间件 / Proxy (Middleware / Proxy) ---> 真实的服务器 (Node.js / Server Components)
```

一般的前端框架在某种程度上存在一个问题：如果用户在浏览器上禁止了 JavaScript 脚本，纯客户端逻辑就很难对请求发送做出有效约束。

Next.js 为了规避这一点，会在服务端请求链路中提供中间件能力，让我们可以在请求真正进入页面或接口之前先做一层判断。

> [!note]
> Next.js 新版文档里已经把 `middleware.ts` 文件约定改名为 `proxy.ts`。旧项目和很多教程里仍然会叫 Middleware / 中间件。本文先沿用“中间件”这个学习语境。

## 何为中间件？🧩

中间件是一段运行在请求进入路由之前的服务端代码。它通常运行在 Edge Runtime 这类更轻量的运行时里，而不是完整的 Node.js Runtime，所以它更适合做快速的请求判断，例如鉴权、重定向、改写 URL、设置请求头等。

它的优势是可以在请求真正抵达页面渲染逻辑或后端接口之前进行拦截。用户不能像修改浏览器里的前端代码那样直接绕过这段服务端逻辑。

## 中间件一般可以做什么？📌

由于中间件处在请求链路前面，它主要能拿到用户发出的网络请求信息，例如：

- 请求 URL；
- 请求头；
- Cookie；
- 查询参数；
- pathname 等路由信息。

需要注意：中间件不适合处理复杂业务逻辑，也不适合读取和处理大体积请求体。它更适合做“轻量、快速、靠请求信息就能判断”的事情。

# 最经典的作用：路由拦截 🚦

前面说过，中间件可以拿到请求头、Cookie 和访问路径，这让我们可以针对路由进行检查。在匹配到对应路径的请求发生时，中间件就会被触发。我们可以通过一些 API 拿到此时的请求信息。

请注意，中间件在拦截请求后，必须告诉 Next.js 下一步要做什么。

## NextResponse

`NextResponse` 是 Next.js 提供的响应辅助 API，它提供了若干个常用方法：

- `NextResponse.next()`：放行这次请求。
- `NextResponse.redirect()`：进行重定向，默认会向浏览器返回临时重定向响应，让浏览器跳转到新地址。
- `NextResponse.rewrite()`：进行请求改写，可以修改实际访问地址，但不会修改浏览器地址栏上显示的地址。
- `NextResponse.json()`：直接返回 JSON 响应。

```typescript
// src/middleware.ts
import { betterFetch } from '@better-fetch/fetch';
import type { auth } from '@/lib/auth';
import { NextResponse, type NextRequest } from 'next/server';

export async function middleware(request: NextRequest) {
  const { data: session } = await betterFetch<typeof auth.$Infer.Session>(
    '/api/auth/get-session',
    {
      baseURL: request.nextUrl.origin,
      headers: { cookie: request.headers.get('cookie') || '' },
    },
  );

  if (!session && !request.nextUrl.pathname.startsWith('/login')) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'], // 需要拦截的路径
};
```

可以看到，中间件会接收一个 `NextRequest` 参数。该参数就是当前请求对象，我们可以通过它拿到请求头、Cookie、访问地址等信息。

## 如何做到路由守卫？🔐

我们虽然可以从请求中拿到 Cookie，但 Cookie 本身并不等于“用户已经登录”。更可靠的做法是拿 Cookie 去后端认证逻辑里验证 session。验证通过就放行，验证失败就重定向到登录页。

上面的代码就是这个思路：

- 从请求头里拿到 Cookie；
- 调用 `/api/auth/get-session` 校验当前 session；
- 没有 session 且访问的不是 `/login` 时，重定向到登录页；
- 已登录或正在访问登录页时，继续放行。




