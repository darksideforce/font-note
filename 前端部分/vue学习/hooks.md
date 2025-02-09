### hooks含义
一般我们所说的hooks是将一些可以复用的代码抽离出来进行抽象以方便复用
一些**可复用的方法**像钩子一样挂着，可以随时被引入和调用以实现**高内聚低耦合**的目标，应该都能算是hook；

### 为什么Vue3要用自定义Hook？：

结论：就是为了让`Compoosition Api`更好用更丰满，让写Vue3更畅快！像写诗一样写代码！ 其实这个问题更深意义是为什么Vue3比Vue2更好！无外呼**性能大幅度提升**，其实编码体验也是Vue3的优点**`Composition Api`的引入（解决Option Api在代码量大的情况下的强耦合）** 让开发者有更好的开发体验。

#### 写Vue3请摆脱Vue2无脑this的思想：

### Vue3的自定义Hook
1. 将可复用功能抽离为外部JS文件
    
2. 函数名/文件名以use开头，形如：useXX
    
3. 引用时将响应式变量或者方法显式解构暴露出来如：`const {nameRef，Fn} = useXX()`
    
    （在setup函数解构出自定义hooks的变量和方法）
``` js
import { ref, watch } from 'vue';
const useAdd= ({ num1, num2 })  =>{
    const addNum = ref(0)
    watch([num1, num2], ([num1, num2]) => {
        addFn(num1, num2)
    })
    const addFn = (num1, num2) => {
        addNum.value = num1 + num2
    }
    return {
        addNum,
        addFn
    }
}
export default useAdd
```

**与vue2mixin的差异**
##### Mixin不明的混淆，我们根本无法获知属性来自于哪个Mixin文件，给后期维护带来困难
``` js

//加法功能-自定义Hook（将响应式变量或者方法形式暴露出来）
const { addNum, addFn } = useAdd({ num1, num2 })
addFn(num1.value, num2.value)
//减法功能-自定义Hook (将响应式变量或者方法形式暴露出来)
const { subNum, subFn } = useSub({ num1, num2 })
subFn(num1.value, num2.value)

```

#### 无法向Mixin传递参数来改变逻辑，但是Vue3自定义Hooks却可以
#### Mixin同名变量会被覆盖，Vue3自定义Hook可以在引入的时候对同名变量重命名

```js
export default {
  mixins: [ addMixin, subMixin], //组件内混入加法和减法Mixin
  mounted(){
      this.add(num1,num2) //调用加法addMixin内部的add方法
      this.sub(num1,num2) //调用减法subMixin内部的sub方法
  }  
}
```