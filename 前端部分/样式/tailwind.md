tailwind是一种原子化的前端css解决方案，他在移动端以及响应式布局方面有优势，方便迅速开发。

## 如何配置自定义tailwind变量？
tailwind虽然有内置多种颜色和数值但是其实还是不够用的，我们基本上都会用到自定义css变量来进行开发。这里以tailwindcssV4来示例。
第一步：我们自然是需要创建好tailwind.config.ts文件

```ts
export default {
  darkMode: "class",
  content: [
  ],
  theme: {
    screens: {
      sm: "640px",
      // => @media (min-width: 640px) { ... }
      md: "768px",
      // => @media (min-width: 768px) { ... }
      lg: "1024px",
      // => @media (min-width: 1024px) { ... }
      // 基础版心
      wrapper: "1200px",
      xl: "1280px",
      // => @media (min-width: 1280px) { ... }
      "2xl": "1440px",
      // => @media (min-width: 1440px) { ... }
    },
    debugScreens: {
      position: ["bottom", "right"],
      ignore: ["dark"],
    },
    extend: {
      color:{
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
      }
    },
  },

};
```
这里有诸多的配置项，我们先来说最常用的颜色变量部分。可以看到theme下有一个子项是extend，在这里我们可以声明一个color对象用来储存即将使用的css变量。注意看background部分。这里他使用了一个css变量`var(--background)`，这就意味着这个background字段和css中的变量建立了映射关系
第二步：创建css中的变量
tailwindcssV4中我们使用私有变量不再需要使用一系列的声明诸如`@layer base`等方式来进行包裹，直接在`:root`下进行声明即可
```css
@import "tailwindcss";

:root {
  --background: 0 0% 100%;
  --foreground: 240 10% 3.9%;
}
```
如图所示，这里我们就是创建了两个变量
第三步：使用
这里就简单了，直接进行使用即可

```jsx
<div className="text-foreground"></div>
```