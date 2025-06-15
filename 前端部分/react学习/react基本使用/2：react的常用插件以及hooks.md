
# hooks

## useState
如名字所示就是

# 插件
tailwind-merge与clsx

### class-variance-authority实现重复的样式管理 #tailwindcss
我们有时候会有一些重复需要定义的样式，如果统一使用tailwind来进行书写会显得麻烦，但是如果用组件实现又会有些笨重，如果我们希望有个折中的方式来统一管理样式，可能会有人想到使用function或者对象来统一管理。但是这里有一个更加高效的插件`class-variance-authority`（以下简称cva）
- 第一步。使用npm install class-variance-authority进行安装
- 第二步。使用`import { cva } from "class-variance-authority";`进行导入
- 第三步。使用
```ts
const buttonVariants = cva(
  "inline-flex items-center justify-center whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-1 focus-visible:ring-ring  disabled:pointer-events-none disabled:opacity-50",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive:
          "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline:
          "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-9 px-3",
        sm: "h-8 px-3",
        lg: "h-11 px-8",
        icon: "size-8 ",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  },
);

```
如上图所示，cva接受两个参数。第一个参数是通用的css类名，第二个参数是可选项。接受两个属性名分别是variants和defaultVariants，代表的是配置和默认配置。如果我们在调用cva时不传入任何选择，则返回的是默认配置中的内容，注意默认配置中的内容是在配置内容中进行查找的。
配置项的格式如下：
- 配置子项：下跟一个类名集合名与一个类名集合，而且需要有一个default代表默认配置
比如如果我这样进行调用
`buttonVariants({ variant: "link" })`这就代表我选中的是variant这个配置子项下的link集合。而size集合则会返回默认的
使用时：
```ts
buttonCva({variant:"link"})
```