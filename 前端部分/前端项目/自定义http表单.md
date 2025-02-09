## 编写form表单

### 如何透传属性并且交给子组件？
vue3支持组件透传属性，当我们给组件名加上各种属性的时候会直接传递给子组件内的最外层元素上
我们可以通过给调用组件时传进一个v-model，这样会直接作用给子组件
但是子组件内部的变量如何接受这个model呢？我们可以通过一个vue3新增api来实现。
`provide`
通过`provide`我们可以把某个变量暴露给所有的子孙组件以供他们调用，该api可以方便某些复杂的子孙组件进行调用。
例如：
`provide('formModel',instance!.attrs.model)`
该行代码的意思就是把model给暴露一个叫formModel的变量以供后代组件使用
`injeft('formModel',undefined)`来进行接收这个变量
如何再循环的子组件内来查找出需要的provide变量呢？

1：获取当前的item在循环内的下标
2：根据下标去获取指定的属性内容
```
const path = computed(()=>props.path??'')
```
3：使用lodash-es的get和set功能，安全的去获取inject得到一些对象



### 一个组件如何接受透传的父组件并进行双向的model
有时候我们只需要在一层父子组件关系中接受外部传入的属性并将其注册成一个双向绑定的变量
以前我们如何做到的？我们一般需要使用update或者一个本地变量接受这个props
但是vue3.4可以使用`defineModel`来进行实现一个简化的写法
在子组件内声明`const model = defineModel()`
在父组件内我们可以直接使用`v-model：model`来操作这个变量



### 如何在一个封装的表单组件内接受外部传入的slot并传入子组件
我们可以通过template+slot并用if的方式进行传入
在调用组件时：
```
<test><template #label></template></test>
```
在封装组件内进行接受并传递给子组件
```
<template #label>
	<slot name="label"/> 当前组件接受外部
</template>
```


### 如何在vue中使用jsx
jsx是一种方便操作和循环的语言来形容html
我们一般可以通过tsx文件内返回一个setup对象来使用，也可以使用setup语法糖`<script setup lang='tsx'>`

如果实在语法糖里使用jsx，记得需要使用`component`组件来进行渲染，类似`<component :is="render">`

### 如何抽象一个from表单？
我们可以通过循环一个对象的方式以类似与低代码形式的方式把表单渲染出来
一般一个表单对象的每个表单子项？：
1：prop-方便对齐属性名
2：label-标签名
3：method-方法
4：required-是否必须
5：type-item内的输入框或者表单组件内的类型，由于某些写法中我们支持直接将组件import后复制给组件
6：非必须：itemprops-一些可以作用到item上的属性名

获知一整个对象的结构后，我们可以开始循环抽象一个from表单组件
1：利用透传的方法， 直接把外部一些属性给透传到第一层组件内部
2：利用row等ui组件，循环第二层组件，构成ui框架
3：在第二层组件，处理好一些类似于select功能的options项，将其处理成以便使用的子组件

实例代码：
使用jsx来实现
```
<template>
	<component :is="render" />
</template>

<script setup lang='tsx'>
import {FromJSXOption} from '@/views/home/module/addDialog'
import {computed,inject} from 'vue'
import {get,set} from 'lodash-es'
const props = defineProps<{
	options:FormJSXoption
}>()
const formModel = inject('formModel',undefined)
const path = computed<any>(()=>{}props.options.prop ?? '')
const filedValue = computed({
	get(){
		return get(fromModel,path.value,null)
	},
	set(value:any){
		if(fromModel && path.value){
			set(fromModel,path.value,value)
		}
	}
})

const sonInnectGenerate = (sonOptions:any) =>{
	let template
	swtich(sonOptions.type){
		case 'a-radio-button':
			template = <a-radio-button value={sonOptions.value} {...sonOptions.attrs}>
				{sonOptions.name}
			</a-radio-button>
			break;
		case 'a-checkbox':
			template = <a-checkbox value={sonOptions.value} {...sonOptions.attrs}>
				{sonOptions.name}
			</a-checkbox>
			break;
	}
	return template
}

const innerGenerate = (options:FormJSXOption) = >{
	let template
	switch(option.type){
		case 'a-input':
			template = <a-input v-model={filedValue.value} {...options.attrs}>
				{
					options.optionsList?.length
				}
			</a-input>
			break;
		case 'a-radio-gropu':
			template = <a-radio-group v-model:value={filedValue.value} {...options.attrs}> 
			</a-radio-group>
			break;
		case 'a-checkbox-gropup'
			template = <a-checkbox-group v-model:value={filedValue.value} {...options.attrs}>
				{options.optionsList?.length?options.optionsList.map(e=>sonInnerGenerate(e):null)}
			</a-checkbox-group>
	}
	return template

}
const render = ()=>
	<a-form-item {...props.options.attrs} model = {filedValue}>
		{innerGenerate(props.options)}
	</a-form-item>

</script>

```
父元素
```
<template>
	<a-form name="formRef" autocomplete="off" @sumbit.prevent :label-col="{span:8}" :wrapper-col="{span:16}">
		<a-row v-for="(item,index) in props.options" :key="`${item.name}_${index}">
			<a-col :span="24">
				<custom-form-template-item :options="converFormOptions(item)>
				</custom-form-template-item>
			</a-col>
		</a-row>
	</a-form>
</template>
<sciprt lang="ts" setup>
import customFormTemplateItem from './'
import {FormOption} from ''
import {getCurrentInstance,provide} from ''
import userForm from 'hook'
const props = defineProps<{
	options:FormOption[]
}>()
const {converFormOptions } = useForm()
const instance = getCurrentInstance ()
provide('formModel',instance.attrs.model)
</sciprt>
```