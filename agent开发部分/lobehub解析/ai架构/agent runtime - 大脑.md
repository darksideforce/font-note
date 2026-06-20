---
title: agent runtime-大脑
aliases:
  - agent runtime
  - 大脑
tags:
  - agent
  - lobehub
  - 架构
status: 待完善
created: 2026-06-03
---

# 大脑
## 基本
所谓大脑其实就是一系列策略模式组成的一套操作指南，它通过接受若干个参数，并返回操作后的一系列参数，它不会有副作用，单纯的输入和输出也不依赖于其他外部的环境。
lobehub中的这套大脑，可以看到本质上是创建了一个switch，通过传入参数的来判断返回的参数，每个switch的case对应的都是ai的一套行为，我也是今天才知道原来对于大脑或者前端或者agent工作原理来说，agent的一段完整的文字输出，直到它抛出其他行为。
所以大致的流程就是这样：
发送信息=》ai回复=〉ai通知需要授权或者调用其他工具=》进行授权=〉ai完成对话=》终止
代码也很简单：
```ts
export class GeneralChatAgent{
   private config: GeneralAgentConfig;
   constructor(config: GeneralAgentConfig) {
    this.config = config;
   }
   async runner({
	   context,
	   state
   }){
    if(state.status === 'interrupted'){
	     return this.handleAbort(context,state)
    }
    swtich(context.phase){
	    case 'user-input':
	    case 'llm_result':
	    case 'tool-result':
	    case 'human_abort':
    }
   }
}
```
我大致简略了一下输出的情况，其实都大查不查没有很复杂的逻辑，都是针对不同的传入方的情况。
- 首先第一步是外部触发user_input：这种通知大脑进行整理历史聊天记录，并返回处理好的数据给外部。
- 模型第一轮回复，提示需要调用工具，并且模型会直接将参数发送过来。
- 大脑收到模型的工具请求后，进行权限和危险级别的判定。
	
	- 危险工具：返回 `request_human_approve` 指令，让外部卡住等用户点同意。
	- 安全工具（或已授权）：返回 `call_tool` 指令，让外部（Runtime）去真正执行跑代码的动作。
	- 注意这些工具请求是并行的，我们需要组装成一个数组进行处理
- 流程终止。

## 大脑内的不同处理流程

我们重点来解析一些这几个步骤：
### 压缩上下文：
一般压缩上下文的做法是如此，我们在大脑内判断用户输入后是否需要压缩上下文
- 接收到用户输入，进行解析上下文。
- 如果上下文超出，则挂起当前聊天，并发起压缩指令，通知大模型进行压缩。

  如果上下文未超出，则发起正常交互，发送信息给上下文。
- 引擎接收到大脑发出的压缩指令后，单独开辟一个请求，要求大模型压缩上下文，并在完成后使用新的上下文进行替换掉旧的上下文
- 当大模型压缩完毕，则返回通知大脑，。

** 为什么要压缩上下文交给ai来做？**
如果单纯的粗暴截取上下文，则会导致上下文可预见的丢失重要信息

**如何实现的替换上下文**
大脑本身不管任何上下文或者产生外部副作用，它只负责将输入的数据处理成某种通行证。
上下文压缩在这里也是同理，大脑本身不负责压缩，也不负责替换上下文。
当一个压缩命令由大脑进行发出后。将由引擎发起一个专门要求llm总结的请求，引擎进行总结后自行替换上下文，大脑完全不用进行关心

**如何判断需要压缩上下文**

### 安全审查方法判断 

大脑的常见触发场景还包含授权方法，接下里讲下如何实现的授权方法 。 #技术亮点
结合我平时使用agent的体验我感觉又这些使用场景是必须覆盖的：
- 阻止ai执行危险的工具类。
- ai需要记住我曾经的授权。
- 授权维度需要很精细。
- 授权肯定包含一些普遍的规则和一些不普遍的规则
- 我们还需要根据用户选择的授权模式来针对不同的授权情况作最后的处理

如果没有授权的步骤，ai将会错误的执行一些奇怪的信息。像删除重要的文件或者针对不同的工具存在不同的权限，所以针对这种的使用情况，需要有灵活的拦截决策来进行拦截才能保证一定的使用体验。
如何设计一个灵活的拦截决策？我们可以这样来进行考虑。
- 首先一定需要有一个全局适配可以应用于所有情况的黑名单系统，这样可以直接拦截掉一些严重的工具授权，但是这一层只能做到简单的拦截工具授权，无法精确的对工具授权内的精确信息进行鉴别。
- 第二层我们就可以设计一个专门的规则引擎来对传入的规则进行检查。(插件化拦截器机制)
- 第三层我们需要更加精细化的适配器，使用一个字典映射器来进行查找，并根据返回值指定对应的通过或不通过

第一二层代码：
```ts
// 1. 抽离出的纯净校验函数（插件）
const blacklistResolver = (toolName, blacklist) => blacklist.includes(toolName);
const rateLimitResolver = (toolName) => false; // 比如未来可能加入的频控拦截
// 2. 将它们注册进一个数组中（组装引擎）
const globalResolvers = [ blacklistResolver, rateLimitResolver ];
// 3. 核心分流方法（双层循环但极其清晰）
function checkIntervention(toolsCalling, blacklist) {
  const toExecute = [], toIntervene = [];
  for (const tool of toolsCalling) {
    let blocked = false;
    
    // 这就是那层保留下来的高级内循环（穿透所有的拦截器）
    for (const resolver of globalResolvers) {
      if (resolver(tool.apiName, blacklist)) {
        blocked = true;
        break; // 只要有一个拦截器返回 true，立马短路退出
      }
    }
    if (blocked) toIntervene.push(tool);
    else toExecute.push(tool);
  }
  
  return [toIntervene, toExecute];
}

```

第三层代码：
该方法是提取出对应的工具配置项（其实就是取出对应的manifest文件）
```ts

 getToolInterventionConfig(
    toolCalling,
    state,
  ):  {
    const { identifier, apiName } = toolCalling;
    // 本地提取到的manifest列表文件
    const manifest = state.toolManifestMap[identifier];

    if (!manifest) return undefined;
	//找到manifest内声明的api和对应的权限要求
    // Find the specific API in the manifest
    const api = manifest.api?.find((a: any) => a.name === apiName);
    // 返回对应的权限申请
    return api?.humanIntervention ?? manifest.humanIntervention;
  }
```
该方法使用配置项进行权限校验，判断是否通过。
```ts
resolveDynamicPolicy(
    config
    toolArgs
    metadata
  ){
    if (!this.isDynamicInterventionConfig(config)) {
      return Promise.resolve(undefined);
    }
    const { dynamic } = config;
    const resolver = this.config.dynamicInterventionAudits?.[dynamic.type];

    if (!resolver) return Promise.resolve(dynamic.default ?? 'never');

    return Promise.resolve(resolver(toolArgs, metadata)).then((shouldIntervene) =>
      shouldIntervene ? (dynamic.policy ?? 'always') : (dynamic.default ?? 'never'),
    );
  }

```

最终如何将工具按自定义方法处理：
```ts

const config = this.getToolInterventionConfig(toolCalling, state);
      const isDynamicConfig = this.isDynamicInterventionConfig(config); //判断是否需要取出配置项
      const dynamicPolicy = await this.resolveDynamicPolicy(config, toolArgs, state.metadata); //经过自定义规则引擎后判断好策略结果
      const staticConfig = isDynamicConfig
        ? undefined
        : (config as HumanInterventionConfig | undefined);

      if (dynamicPolicy !== undefined) {
        if (dynamicPolicy === 'never') {
          toolsToExecute.push(toolCalling);
        } else {
          toolsNeedingIntervention.push(toolCalling);
        }
        continue;
      }

```

### 安全审查责任链流程图

![[Excalidraw/Agent 工具拦截与授权分流.flowchart.md]]

**最终的调用顺序**

1. 创建两个数组：一个存放需要人工审批的工具，一个存放不需要人工审批、可以直接执行的工具。
2. 取出全局黑名单，并组装好本轮审查需要使用的 `metadata`。
3. 取出当前用户的授权模式。
4. 取出全局默认规则引擎。
5. 开始针对所有待审核工具逐个审查。
   1. 为当前工具创建两个临时状态：全局 `policy` 和全局禁用状态。
   2. 经过全局规则引擎检查后，如果有任一规则命中，就立刻中止全局规则引擎的后续检查，将禁用状态改为 `true`，并把 `policy` 改为需要审查。
   3. 如果该工具被全局规则命中，且返回结果为需要审查，则立刻推入人工审批数组。
   4. 如果工具配置了自己的动态规则，则使用动态规则进行检查：需要审查的推入人工审批数组，不需要审查的直接放入可执行数组。
   5. 如果工具被全局规则引擎命中，但结果不是“需要审查”，而是其他策略判断，也统一塞入人工审批数组，由外部继续处理。
   6. 检查该工具是否在配置文件里写了静态 `humanIntervention` 审查要求。如果有，也塞入人工审批数组。
   7. 检查当前模式是否为自动模式。如果是自动模式，则立刻放行。
   8. 如果触发了幻觉工具调用，且当前不是自动模式，则需要人工审批。
   9. 如果用户选择的档位为白名单模式，则判断该工具是否在白名单内：在白名单内则放行，不在白名单内则塞入人工审批数组。
   10. 如果用户选择的档位为手动模式，则按照工具的静态规则进行最后一遍处理。
11. 将处理好的两个数组返回给外部。

## 工具返回结果，调用次级ai
 #待补充如何实现 


## 处理工具类的执行结果
有一个一定会出现的场景，就是当ai通知大脑需要有若干的工具进行授权。但是人类的授权肯定是一次一次进行授权的，这会出现一个情况。如果我们在每次授权后都发送信息给模型，模型肯定会出现问题。这里就要提到一个情况。
**模型的调用工具的回复，是一口气发送过来的，也就是说如果需要调用3个工具，它会在一次会话中直接发送过来**
所以，同样的，我们需要一次性将所有工具的调用结果也给返回到大模型侧。这就延伸出很多场景：
- 假如用户是过了很久才授权下一个工具呢？
- 假如用户拒绝了其中的一些工具呢？

lobehub的做法是，通过两个设计模式：批发审批和静默拒绝
在进行审批时，使用的是批发审批
在进行发送结果时，使用的是静默拒绝。
```ts
// 取出当前仍在等待审批的工具
const pendingToolMessages = this.getCurrentTurnPendingToolMessages(state);
	//进行一次数据过滤并处理好交给外部
        if (pendingToolMessages.length > 0) {
          const pendingTools = pendingToolMessages.map((m: any) => m.plugin).filter(Boolean);
			//通知引擎进行静默等待，知道下一次引擎通知大脑
          return {
            pendingToolsCalling: pendingTools,
            reason: 'Some tools still pending approval',
            skipCreateToolMessage: true,
            type: 'request_human_approve',
          };
        }
        //当没有需要等待的审批的数据时，直接交给引擎
        return this.toLLMCall(
          {
            messages: state.messages,
            model: this.config.modelRuntimeConfig?.model,
            parentMessageId,
            provider: this.config.modelRuntimeConfig?.provider,
            tools: state.tools,
          } as GeneralAgentCallLLMInstructionPayload,
          state,
        );
```




# 执行层

# 如何自我循环

# 如何中断？

# 告诉ai有哪些待办事项

# 如何处理人类授权

# 如何处理报错
