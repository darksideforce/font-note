---
aliases:
  - react传递事件
tags:
  - react
  - 组件封装

status: 待完善
created: 2026-06-14
---


# react和vue有何不同
在vue中，我们需要使用发布订阅模式来实现事件的传递，在子组件内声明好传出的事件，并在父组件内通过监听自定义事件的方式来进行交互。
但是在react里截然不同，react完全遵循函数式编程的精髓，不会过多设计api，父组件与子组件交互的方式是通过将一个回调函数传递给子组件的方式来进行监听。
在react的眼中，事件监听器，比如onclick等其实和其他的props属性是一模一样，没有任何区别，react通过传递一个js的函数来作为props属性传递，子组件执行js函数，就可以达到执行外部设备父组件的回调了。


# 那爷孙通信呢？
vue提供了provider和reject来作为爷孙组件的传递，react则使用contextProvider作为爷孙通信，比vue的方式要稍微麻烦一点。
父组件暴露出一个contextProvider，并注入变量
```tsx
const ChatContext = createContext<ChatContextType | null>(null)
<ChatContext value={{ onClearChat: handleClear }}>
 <MiddleContainer />
</ChatContext>

```
孙组件调用use来消费变量
```tsx
import ChatContext from ''
const context = use(ChatContext);
```
需要注意的是这个chatcontext变量需要被导出而且不能设置在组件内部，它需要被孙组件给引入并消费。而且需要注意的是每次这个变量的值改变都会触发所有消费该变量的组件的重新渲染。
**之所以这个变量不能在组件内部**
是因为如果在组件内部的话，组件每次的渲染都会导致这个变量重新改变，导致严重的 问题
react这个hooks在底层是利用全等以及内存来进行判断，如果内存地址改变或者值改变都会触发重新孙组件和爷组件的重新连接

# 案例：
```tsx
import React from 'react';

interface IconButtonProps extends React.ComponentProps<"button"> {
    children: React.ReactNode;
}

export default function IconButton({ children, ref, className = "", ...props }: IconButtonProps) {
    return (
        <button
            ref={ref} // 3. 将 ref 绑定到真实 DOM 上，供 Dropdown/Popover 计算定位
            {...props} // 4. 将所有隐式注入的事件（onClick, onMouseEnter 等）一次性展开绑定
            className={`
                ${className}
            `}
        >
            {children}
        </button>
    )
}


```