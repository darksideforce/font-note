react的基本使用相当简单，我们只需要使用npm进行下载react即可。接下来需要注意一下流程
+ 首先页面中进行引入react和reactdom以及babel进行解析
+ script标签中使用`<script type="text/babel">`进行解析
+ 创建一个元素并设置好唯一标识以便让react进行捕捉
+ 使用`const root = ReactDOM.createRoot`来进行创建react页面，参数为可被捕捉的元素
+ 在创建好后，调用`render`方法进行渲染，render的参数为一个jsx元素

类组件：
+ 首先需要使用`class App extends React.Component`来进行继承react元素的组件
+ 在`constructor`声明好大部分的属性，并使用super来继承`React.Component`
+ 所有的声明式变量都需要在state内声明好
+ 如果需要改变state则需要使用`this.setState`来进行声明，且需要制定修改的变量

```js
        class App extends React.Component{
            constructor(){
                super()
                this.state = {
                    message:['小美','小王','小姜','小铁'],
                }
            }

            render(){
                return (
                    <div>
                        <ul >
                            {this.state.message.map(e=> <li>{e}</li>)}
                        </ul>
                    </div>
                )
            }
        }
        const root = ReactDOM.createRoot(document.querySelector('#root'))
        root.render(<App/>)
```

-----
### jsx的基本语法
+ jsx的最上层元素只能有一个
+ 在jsx内使用注释可以使用`{/*注释*/}`来进行使用
+ jsx内插入数据无法插入对象类型，可以插入string，number和array类型，null和undefined和boolean会为空
+ 绑定元素属性直接使用大括号语法即可`<div title={title} />`
+ 绑定元素类名使用`<div className={className}>`
+ 绑定style使用`<div style={ {color:'red'} }>`
