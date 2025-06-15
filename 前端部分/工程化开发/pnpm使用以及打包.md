## 简介
现在的大部分开源项目都是使用monorepo的方式来进行构建项目，一般monorepo是使用pnpm来进行打包。
Pnpm有几个个比较和npm不一样的地方
- 他拥有一个packages文件夹，下面是各种项目，而每个项目拥有自己独立的nodemodules，使用pnpm安装依赖的时候我们可以选择安装到具体那个项目的依赖
- pnpm会将依赖安装到系统层面或者全局的储存位置。通过一些索引来进行访问，npm会使用扁平的依赖结构
如果我们想将所有的依赖按npm的方式来进行安装，可以在`.npmrc`文件内加入`shamefully-hoist=true`

## 开始准备打包
现在开始学习如何使用pnpm搭建一个项目以及使用相关
- 根目录下创建pnpm-workspace.yaml文件，这是pnpm的一些配置文件，我们在内部加入
```
packages:
  - 'packages/*'
```
表示项目全在packages目录下
- 创建打包脚本。根目录下创建scripts文件夹-dev.js文件，并在package.json内加入打包命令
- packages下不同的项目使用不同的打包命令，比如packages下的shared项目使用该打包命令
```
"scripts":{
	"dev":"node scripts/dev.js shared"
}
```
- 我们使用esbuild来进行打包，node中的命令通过process.argv + minimist来进行获取打包参数
```ts
import {resolve, dirname} from "path";
import {fileURLToPath} from "url";
const args = minimist(process.argv.slice(2)); //node中的参数通过process来获取
const __filename = fileURLToPath(import.meta.url); //获取当前文件的绝对路径
const __dirname = dirname(__filename);  //获取当前文件的目录
const entry = resolve(__dirname, `../packages/${target}/src/index.ts`); //获取入口文件的绝对路径
```
- 因为我们统一使用打包脚本来进行打包，所以每个项目最好格式都类似，拥有一致的入口文件和目录结构
- 模块/项目下的依赖的package.json文件是描述一个项目的地方，可以在此描述
-  入口文件也在模块/项目package.json中进行描述，像module（入口文件）、unpkg（cdn入口文件）在此进行声明 
- 如果我们需要将一个同项目下的包安装到另一个项目内。使用该命令
`pnpm install packageA --workspace --filter packageB`
--workspace指的是当前命令式在工作区范围内进行的，如果不加--filter则是安装到根目录下
--是进行你可以通过它来选择特定的一个或多个工作区，指的是命令针对的是某个项目工作区内
- 在scripts文件夹-dev.js文件内开始进行打包
```ts
import minimist from "minimist";
import {resolve, dirname} from "path";
import {fileURLToPath} from "url";
import esbuild from "esbuild";
import { createRequire } from "module";

//node中的参数通过process来获取
const args = minimist(process.argv.slice(2));
const __filename = fileURLToPath(import.meta.url); //获取当前文件的绝对路径
const require = createRequire(import.meta.url);
const __dirname = dirname(__filename);  //获取当前文件的目录
const target = args._[0] || "reactivity"; //获取打包目标
const format = args.f || "iife"; //获取打包格式
//获取有关项目的信息
const entry = resolve(__dirname, `../packages/${target}/src/index.ts`); //获取入口文件的绝对路径
const pkg = require(resolve(__dirname,`../packages/${target}/package.json`))//获取项目的打包配置项
//进行打包
esbuild.context({
    entryPoints:[entry],//入口文件
    outfile:resolve(__dirname,`../packages/${target}/dist/${target}.js`),//出口文件
    bundle:true,//是否集中打包
    sourcemap:true,//是否支持调试
    format,//打包格式
    globalName:pkg.buildOptions.name,

}).then((ctx)=>{
    console.log("watching...");
    return ctx.watch();//开启持续化打包
})

```