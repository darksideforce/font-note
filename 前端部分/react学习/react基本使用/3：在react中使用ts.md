#react #ts
# 组件中


## 组件通信 

一： 如果我们需要使用一些ts方法声明一个组件的某些接口与其他类型类似，可以使用
`React.ComponentProps`工具函数来进行继承等操作
```ts
export type NextLinkProps = React.ComponentProps<typeof Link>
//这表示他讲把尖括号内的一些ts类型转换为可被组件接受的props类型
```
这样可以把诸如children和classname等属性默认和link的属性进行合并


# 接口调用