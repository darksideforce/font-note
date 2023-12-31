<a name="PnVMy"></a>
# ts基本使用
ts就是js的一个超集，ts需要编译器来进行编译才能进行。<br />使用
```javascript
npm install -g typescript
```
来进行安装ts，ts文件不被游览器所支持，如果ts文件中只有js语法，是不会报错的。<br />如果ts文件中有ts语法，就需要使用ts来把这个文件编译成js文件才可以继续进行。

<a name="VI2AM"></a>
## 如何通过vscode自动编译ts文件	
1：使用tsc --init来生成一个ts的配置文件<br />2：在配置文件中找到 "outDir":"./",选项。这是设置编译后的文件存放的位置。<br />3：在配置文件中"strict":false,选项，这是是否开启严格模式<br />4：打开vscode，选择终端，选择任务，选择所有任务，选择监听<br />![image.png](https://cdn.nlark.com/yuque/0/2022/png/1733768/1641888098731-7f5edb99-78ff-42c9-b034-07b86d946aa8.png#averageHue=%232b383f&clientId=u7f76345f-827f-4&from=paste&height=180&id=u044e42e4&originHeight=180&originWidth=586&originalType=binary&ratio=1&rotation=0&showTitle=false&size=17175&status=done&style=none&taskId=u9079c23a-39ad-4fa2-b3c0-5a13355d9b7&title=&width=586)<br />这样vscode就会自动把ts文件编译到指定的文件夹位置了。


<a name="nIDpu"></a>
# ts的类型
ts的类型检测也可以放在形参内。也可以用来规定函数的返回值
```javascript
function sum(a:number,b:number):number{
	return a + b
}
//这个函数的返回值被限定为number类型 
```
布尔值
```javascript
let isDone: boolean = false;
```
any：可以接受任何类型的赋值，同时如果声明变量时没进行任何类型指定，也是进行any类型赋值。
```javascript
let d
let d:any
//这两个等价
```
unknown：表示未知类型的值。
```javascript
//any类型的值可以被付给任何变量。这会导致一些错误。
//但是unknown不允许
let e :unknwon
e='hello'
let s:string
s = e//报错unknown不与其他类型相等
```
类型断言
```javascript
s = e as string
//告诉解析器变量的实际类型
```
void和never<br />void指的是空值，never指的是没有值

object指的是对象，一般来说针对变量进行类型检测，都是需要检测这个变量的属性。所以可以通过大括号的方式来指定object的属性检测<br />T

<a name="yZ5sI"></a>
## Ts的接口
接口是一种对于变量的约束，当我们需要传入的是一个对象的时候，如何对对象的每个属性来进行约束？<br />答案是使用interface，接口<br />这样就限制了传入的对象必须是含有Iperson接口的结构，并且数据类型也需要一致<br />多个接口声明作并集处理。<br />如果要求有额外的自由属性，可以使用[propName:string]:any来进行处理。<br />如果一个属性可选可不选，但仍需强要求类型，可以使用？来表示可选可不选<br />interface可使用extends 来进行继承
```javascript
(()=>{
	interface Iperson{
		firstName:string
  	lastName:string
    [propName:string]:any//这里的类型会影响其他属性，
    age?:name//可选可不选的值
	}
  function showFullName(person:Iperson){}
})
```
<a name="e6X7M"></a>
## TS的类
ts中可以实现类似于java中的类
```javascript
(()=>{
	interface Iperson{
		fisrstName:string,
    lastName:string
	}
  class Person{
  	firstName:string
    lastName:string
    fullName:string
    //构造器函数，用于初始化数据
    constructor(firstName:string,lastName:string){
    	this.firstName = firstName
      this.lastName = lastName
      this.fullName = lastName+fisrstName
    }
  }
})()
```
constructor方法用来创建和初始化对象，把外部传入的值<br />如何使用类来创建新对象呢
```javascript
const person = new Person('诸葛','孔明')
```
这一步就创建了一个属于Person类的对象

<a name="UYPvJ"></a>
## TS的数组类型
```javascript
1:let arr:boolean[] 
2:let arr:Array<boolean> //泛型方式
3:interface x{
  name:String
}//interface方式
let  arr:x[] = [{name:'123'}]
4:多维数组
let arr:number[][]
```
<a name="uFheA"></a>
##  函数重载
当有数个函数声明时，可以根据形参的不同，规范不同的返回值，使用函数时会根据形参的类型来自动去匹配合适的函数声明
```javascript
function fn(params:number):void//重载函数声明
function fn(params:string,params2：number):void//重载函数声明
function fn(params:any,params2?:any):void{
//执行函数  
}
//当我们使用fn函数时，会根据入参的不同自动去匹配2个重载函数中的一个

```

<a name="AvK7E"></a>
### 类型声明的几种方式
联合类型，类似于或，使用 | 来进行链接<br />交叉类型，类似于与   使用&来进行链接<br />类型断言，当我们使用了联合类型后，容易与本身语法冲突。可以使用类型断言，规避编译器的错误。但仍会出发v8引擎的错误
```javascript
let fn = function(num:number | string):void{
  return num.length
}
//上面这串代码会在ts编译器中报错，因为number将不会有length这个属性
//使用类型断言
let fn = function(num:number | string):void{
  return (num as number).length
}
```
<a name="cZxKa"></a>
### ts的类
ts的类变量有几种类型方式<br />1：public 公共的 类的属性可以在外部访问到<br />2：private 私有的 类的属性不可以在外部访问到，也不能由子类访问到<br />3：static 静态的，不使用new来创建类的实例也可以访问到的属性 , 静态的方法也无法访问到内部变量<br />4：protected 保护的 不可以由外部访问到，但是可以由子类访问到
```javascript
class Person{
  protected name :any
  private age :any
  public sub :any
  static aaa :any
}
```
如何通过接口规范类？<br />使用implements关键字
```javascript
class man implements Person{
  
}
```

<a name="Ae4fy"></a>
### ts的枚举
枚举类似对象，内部是默认从0开始的一串属性
```javascript
enum color {
  red,
  green
  blue
}
console.log(color.red) //0
//枚举也可自增
enum color {
  red = 5,
  green,
  blud
}
console.log(color.green)//6 
```

<a name="EjVYg"></a>
### ts中的泛型
泛型是方便接受类型的一种写法，定义类型或者函数的时候，不决定入参的类型，而是由调用方决定.<br />语法为函数名字后面跟一个<参数名>，当使用该函数时把参数的类型传进去即可
```javascript
function add<T>(num:T):Array<T>{
  return [num]
}
//调用
add<number>(1)
//也可以使用类型推断
add(1)
//如果有多个参数
function add<T,U>
add<number,string>(1,'1')
```
在使用泛型时仍会出现问题，因为泛型不带任何属性，所以ts容易报编译错误，使用泛型约束可以避免该问题
```javascript
interface len{
  length:number
}
//泛型约束，使用extends即可涵括新属性
function getLenth<T extends Len>(arg:T){
  
}
```
<a name="SdtnV"></a>
# ts的几种语法
```javascript
function demo(a:number,b?:number):number{}
```
a的number表示的是该传入值必须是number类型<br />b的？表示该值不是必选项<br />括号后的number表示该函数的返回值是number类型。

---

当指定的变量本身就具有类型时，会作类型推导。导致值本身就具有类型。
```javascript
let a = '123'
```
此处的a会因为类型推导默认被赋string类型。函数的返回值同样类型。

---

如果要指定一个数组全是number类型。可以使用
```javascript
const arr:number[] //一维数组
const arr:number[][] //二维数组
const arr:[number,number,number]//元组。指定成员的类型和个数
```

---

如果需要指定一个变量的类型为多种,使用 | 进行分割
```javascript
let color:string|number
```
这种方法也可以进行限制变量的取值
```javascript
let color:'famale'|'male' //这指的是必须选取female和male中的一种
```
<a name="nv6Pi"></a>
# 接口
用于表示对象的几种属性，每个属性的类型必须是什么
```javascript
interface obj {
  name:string;
  id:number
}
```
<a name="DRmai"></a>
# 函数签名
用于限制某个函数必须有特定的参数和返回值。
```javascript
function getName(callback:(data:string)=> void){}
```
这里表达这个函数必须接受一个回调函数。并且这个回调函数接受一个字符串作为签名并且返回空
