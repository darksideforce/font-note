AST即抽象语法树，是一种把js抽象成对象层的表达方式，通过把一段js代码解析成不同片段的字符串和对象来表达一个代码，通俗概念就是ast对象就是一个深度嵌套的对象，来描述一段代码内的所有信息。

<a name="nrvVm"></a>
# 语法
假设有<br />**var** a = 1<br />这样的代码，将其转换为ast格式以后，就成为一个json格式
```json
{
  "type": "Program",
  "sourceType": "script",
  "body": [
    {
      "type": "VariableDeclaration",
      "kind": "var",
      "declarations": [
        {
          "type": "VariableDeclarator",
          "id": {
            "type": "Identifier",
            "name": "a"
          },
          "init": {
            "type": "Literal",
            "value": 1
          }
        }
      ]
    }
  ]
}
```
其中的type是具有特定格式的<br />![](https://cdn.nlark.com/yuque/0/2022/png/1733768/1670291972886-c1cd74bf-e1d4-4985-8f14-8fb80d045d7b.png#averageHue=%23fafdfd&clientId=u532c43fe-1f82-4&from=paste&id=u4c2399ac&originHeight=810&originWidth=1458&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=ub5ca5edc-b622-45ff-929a-6b2d79e8f62&title=)<br />![](https://cdn.nlark.com/yuque/0/2022/png/1733768/1670291983697-55921603-f6ef-471f-bf69-ecb3c96304a5.png#averageHue=%23e2e2e2&clientId=u532c43fe-1f82-4&from=paste&id=uf3adb1eb&originHeight=976&originWidth=1205&originalType=url&ratio=1&rotation=0&showTitle=false&status=done&style=none&taskId=uc82d03f2-b758-4018-91c9-cd7a5754b90&title=)<br />我在这里是用的是recast来进行的ast语法解析。<br />使用recast的visit来进行遍历属性
```javascript
recast.visit(innerAST, {
		visitImportDeclaration: function(path) {
		  const node = path.node
		  nodeArray.push(node)
		  this.traverse(path)
		}
	  })
```
recast的visit是通过遍历ast节点来输出数据。并触发多次回调<br />常见的ast可以参考该文档：[ast节点参考](https://blog.csdn.net/weixin_40906515/article/details/118004822)
