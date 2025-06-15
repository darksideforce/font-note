## 简介

reactivity的核心部分是`reactive`和`effect`

## reactive外层
我们都知道reactive实质上是相当于一个proxy对vue对象进行的一个代理，通过get和set来对proxy来进行一个拦截。
-我们如果需要访问一个属性，实际上是命中了vue对象的get方法，即使我们访问的属性也会命中他的get方法
-如果我们想修改他的属性或者值，实际上也是命中了他的set方法， 即使我们输入的值不存在。
通过这两个拦截，我们就可以将一个数据的变化来通知视图层

 **get和set的区别**
 get：收集副作用，递归的将一个对象内的所有属性处理成响应式
 set：触发副作用并更新值

---
我们先来实现使用proxy进行拦截的这一步。
1. 如果使用vue3的方式来进行代理一个对象，我们先需要一个出口来让用户进行使用
```ts
export function reactive(target){
	return createReactiveObject(target)
}
```
之所以我们需要将代理的步骤给抽离出来，是因为shallowReactive等接口还用到了reactive，所以需要进行抽离
2. 接下来我们还需要处理一下传入的是否为对象，因为在vue3中对象和基本数据结构是通过不同方式来进行处理的
```ts
function createReactiveObject(target){
	if(!isObject(target)){
		return target
	}	
}
```
3. 接下来我们就开始处理一下关于set和get部分
首先接下来复习一下proxy，一个proxy最少需要一个target（目标对象），一个handler（代理函数）,代理函数可以代理一个函数的has，get，set，deleteProperty等行为
```ts
const proxy = new Proxy(target, handler);
const handler = {
//set和get方法会传递几个形参
	set(target,key,value,receiver){ 
	},
	get(target,key,receiver){
	}
}
```
target：被代理对象本身
key：属性名
receiver：代理对象本身，通常为proxy对象本身
value：新值
4. 现在通过函数代理vue3的响应式
```ts
let mutableHandlers = {
	get(target,key,receiver){
		return value
	},
	set(target,key,value,receiver){
		
	}
}
function createReactiveObject(target){
	if(!isObject(target)){
		return target
	}
	let proxy = new Proxy(target,mutableHandlers)
	return proxy	
}
```
5. 现在处理一些参数问题，当传入的如果为proxy对象或已经处理过的对象时，如何节省性能
```ts
//一般处理已经处理过的对象就使用weakMap来进行缓存,每次并在创建过程中判断weakMap是否存在target的键，若不存在则以target为键存入
const reactiveMap = new WeakMap()
//我们可以通过其他方法来处理传入对象是否为proxy的方式，因为即使proxy对象不存在某属性，我们对不存在属性的访问也会被get拦截，所以可以用get来处理
let mutableHandlers = {
	get(target,key,receiver){
		if(value === '_V_isReactive'){}
	}
}
function createReactiveObject(target){
	if(!isObject(target)){
		return target
	}
	if(target._V_isReactive){ //如果传入的不是proxy则不会触发get方法，如果是则会触发get方法，这一步甚至不需要对象有_V_isReactive属性
		return target 
	}
	const existProxy = reactiveMap.get(target)
	if(exitsProxy){
		return exitsProxy
	}
	
	let proxy = new Proxy(target,mutableHandlers)
	reactiveMap.set(target,proxy)//每次都将target和对应的proxy传入weakMap
	return proxy	
}
```
 6. 需要注意的是需要额外注意一些场景，一些特定的场景会导致get拦截器无法正确获取到对象的this指向，所以我们需要使用reflect来进行代理
 **为什么要使用reflect？** #高阶js #reflect
当使用proxy代理一个具有访问器属性的对象时，如果不使用reflect会出现this作用域指向失败的问题。见图所示
![[源码部分/手写vue/assets/Pasted image 20250310174857.png]]
```ts
const person = {
	name:'my',
	get alias(){
		return this.name + 'hello'
	}
}
const proxy = new Proxy{
	get(target,key){
		console.log(key)
		return target[key] //这一步将会被忽略，因为我们在这里访问了target却并没有触发proxy自身的get
	}
}
console.log(proxy.alias)
//alias
//myhello
```
如果我们不使用reflect来进行修饰，这里的步骤将是 1.进入proxy的get，打印出key 2.进入person的访问器，打印出myhello。
问题就出在这里，因为我们在proxy最后一步还是访问了proxy的get，,也就是alias中的this.name这一步。但最后一步却并没有被执行。应该打印出一个name
```ts
const person = {
	name:'my',
	get alias(){
		return this.name + 'hello'
	}
}
const proxy = new Proxy{
	get(target,key,recevier){
		console.log(key)
		return Reflect.get(target,key,recevier)
	}
}
console.log(proxy.alias)
//alias
//name
//myhello
```


## effect原理
