日常项目中，我们通常会出现一些组件需要在很多地方使用，如果一个个对其进行引入到app.vue中会出现繁琐的问题，我们可以通过使用一些vite的插件来实现自动引入

插件部分：
使用`## unplugin-vue-components`该插件来实现自动引入化项目，支持vite
在viteconfig.js中进行引入该插件
```js
import Components from 'unplugin-vue-components/vite' 
export default defineConfig({ 
	plugins: [ 
		Components({ 
			/* options */ 
			}), 
		], 
	})

```
options配置项如下：
```js
	resolver:内置了一些ui框架的解析器
	dirs：输入目录
	dts：输出目录
	directoryAsNamespace:布尔值，是否开启命名空间
```

如何使用svgicon？
svgicon也是一项可以自动导入svg图片的方法，可以节省每次导入图片的人力
步骤如下：
1：下载插件

```js
npm i vite-plugin-svg-icons -D
```
2：在main.js或main.ts中进行导入
```js
import 'virtual:svg-icons-register'
```
3：在viteconfig.js中进行配置
```js
import { defineConfig } from "vite"; 
import vue from "@vitejs/plugin-vue";
import { createSvgIconsPlugin } from "vite-plugin-svg-icons"; 
import { resolve } from "path"; 
const pathSrc = resolve(__dirname, "src"); 
export default defineConfig({ 
	plugins: [ 
		vue(), 
		createSvgIconsPlugin({ 
		// 指定需要缓存的图标文件夹 
		iconDirs: [resolve(pathSrc, "assets/icons")], 
		// 指定symbolId格式 
		symbolId: "icon-[dir]-[name]", }),
	], 
	resolve: { 
		// 设置别名 
		alias: { '@': resolve(__dirname, resolve(__dirname, "./src")) } 
		}, 
	});

```
4：创建svgicon.vue文件
```vue
<template>
  <svg aria-hidden="true" :fill="color" :style="'width:' + size + ';height:' + size">
    <use :xlink:href="symbolId" />
  </svg>
</template>

<script setup>
import { computed } from "vue";
const props = defineProps({
  // icon 名字
  name: {
    type: String,
    default: "",
  },
  // 填充颜色
  color: {
    type: String,
    default: "black",
  },
  // 大小
  size: {
    type: String,
    default: "1em",
  },
});
const symbolId = computed(() => `#icon-${props.name}`);
</script>
```
5：在对应的文件中进行使用
```vue
<template>
  <div class="content">
    <SvgIcon name="client" size="10rem" />
    <SvgIcon name="client" size="10rem" color="red" />
    <SvgIcon name="client" size="10rem" color="green" />
  </div>
</template>
<script setup>
import SvgIcon from "@/components/SvgIcon/index.vue";
</script>
<style lang="scss" scoped></style>
```