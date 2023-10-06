<a name="3luEf"></a>
#### 一：图片懒加载
图片懒加载的原理就是进入项目或者当前页面时，图片并不会同一时间加载出来，而是随着用户的下滑等操作慢慢加载<br />步骤如下：<br />第一步：给所有需要懒加载的图片元素（即image标签）不设置src属性，然后把原本需要指向的src值设置成一个类名（如lazy-load）。这有2个作用<br />1：为了以后获取需要懒加载的元素<br />2：可以设置一个加载中的图片<br />第二步：获取所有类名为懒加载的元素，并循环判断是否在可视区域内。如果在可视区域内，就把图片的src地址设置成真正的地址
```javascript
inViewShow() {     
    let imageElements = Array.prototype.slice.call(document.querySelectorAll('.lazy-image'))    
    let len = imageElements.length     
    for(let i = 0; i < len; i++) {         
        let imageElement = imageElements[i]        
        const rect = imageElement.getBoundingClientRect() // 出现在视野的时候加载图片         
        if(rect.top < document.documentElement.clientHeight) {             
            imageElement.src = imageElement.dataset.src // 移除掉已经显示的             
            imageElements.splice(i, 1)             
            len--             
            i--         
        }     
    } 
}
```


<a name="jrnp6"></a>
## 二：switch语句优化
可以把switch的判断写成对象的属性和变量名，再有一个专门的函数进行处理，即可代替简单的switch语句
```javascript
jumpSwitch(page, len, array) {
      const actions = {
        'editSentenceGroupPage': ['settingThemeData/SET_CHANGESENTENCEGROUP', array],
        'editNaturalSpellingPage': ['settingThemeData/SET_CHANGEEDITLIST', array[0]],
        'editHighFrequencyWordPage': ['settingThemeData/SET_CHANGEHIGHWORDLIST', array]
      }
      const onPageJump = (status) => {
        const action = actions[status]
        this.$store.commit(action[0], action[1])
        this.pageNext(page, len)
      }
      onPageJump(page)
    },
 //上面的代码和下面的代码同效果，不仅优化了代码量，还便于维护，减少系统出错可能性
 switch (page) {
        // 句型组
        case 'editSentenceGroupPage':
          this.$store.commit('settingThemeData/SET_CHANGESENTENCEGROUP', array)
          this.pageNext(page, len)
          break
        // 自然拼读
        case 'editNaturalSpellingPage':
          this.$store.commit('settingThemeData/SET_CHANGEEDITLIST', array[0])
          this.pageNext(page, len)

          break
          // 高频词
        case 'editHighFrequencyWordPage':
          this.$store.commit('settingThemeData/SET_CHANGEHIGHWORDLIST', array
          )
          this.pageNext(page, len)
          break
      }
```

<a name="GsZHh"></a>
## 可用的节流器
```javascript
debounce (fn) {
      let canRun = true;
      return function () {
        // 5、在函数开头判断标志是否为 true，不为 true 则中断函数
        if (!canRun) {
          return;
        }
        // 6、将 canRun 设置为 false，防止执行之前再被执行
        canRun = false;
        // 7、定时器
        setTimeout(() => {
          fn.call(this, arguments);
          // 8、执行完事件（比如调用完接口）之后，重新将这个标志设置为 true
          canRun = true;
        }, 1000);
      };
    },
```
<a name="mnsfz"></a>
## 回车刷新问题
当页面只有一个form表单触发项时，使用回车会导致页面刷新
```javascript
<el-form :model="queryParams"
               ref="queryForm"
               :inline="true"
               v-show="showSearch"
               @submit.native.prevent
               label-width="68px">
        <el-form-item prop="name">
          <el-input v-model="queryParams.name"
                    placeholder="请输入关键字查询"
                    clearable
                    size="small"
                    @keyup.enter.native="handleQuery" />
```
使用阻止原生的提交行为可以阻止该刷新行为

<a name="fjkoA"></a>
## 如何动态引入js文件
```javascript
loadJsAsync(src, async, options){
      return new Promise((resolve, reject) => {
        const script = document.createElement("script");
        script.src = src;
        script.async = async;
        if (options) {
          for (const key in options) {
            script.setAttribute(key, options[key]);
          }
        }

        const onload = () => {
          console.info("js loaded: ", src);
          script.removeEventListener("load", onload);
          
          resolve();
        };

        script.addEventListener("load", onload);
        script.addEventListener("error", (err) => {
          script.removeEventListener("load", onload);
          console.error("loading js error: ", src, err);
          reject(new Error(`Failed to load ${src}`));
        });

        (
          document.getElementsByTagName("head")[0] || document.documentElement
        ).appendChild(script);

      });
    },
```
注意需要在mounted中动态引入
