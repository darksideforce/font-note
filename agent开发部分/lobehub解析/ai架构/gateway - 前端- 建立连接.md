---
title: client - 建立 WebSocket 连接
aliases:
  - client
  - 建立 WebSocket 连接
tags:
  - agent
  - lobehub
  - sse
  - gateway
status: 待完善
created: 2026-06-06
---

# 🔌 连接是什么？为什么要建立连接？

**架构背景**：client 路由和 server 托管是两种截然不同的和大模型进行通信的方式，LobeHub 这种开源热门的项目自带支持纯前端直连（使用自己的 API Key）和后端托管的方式进行连接。现代成熟的 Agent 基建大多都是通过后端托管的方式（gateway）来与大模型进行通信，目的包括但不限于有统一鉴权、Token 计费拦截。也方便拓展类似于 RAG、持久化记忆检索以及私有化知识库等。

但是这样也带来一个问题。当我们把重逻辑都放在后端时，在 Agent 这种复杂逻辑以及每次通信时间极长的请求时会出现交互问题，前端应该如何稳定地和后端的 Agent 进程进行交互。

**解决办法**：封装独立的长连接管理器。建立基于 WebSocket 的单向数据订阅通道。实时捕捉后端 Token 与工具执行状态。提供异常中断方法，及时通知后端杀死 Agent 执行线程。

# 📌 前置知识

## 🧠 数据缓存与恢复

使用 WebSocket，肯定会有时间延迟，当我们发起 WebSocket 连接后，立马后端就向大模型开始请求数据并立马进行输出。但是此时我们仍然需要向后端发送信息来进行鉴权等前置行为。这短时间内可能就会错过后端很多的数据。

如何来进行规避呢？

**我们可以遵循这样的链路**：

- 发起 WebSocket 连接，通知后端向大模型进行连接
- WebSocket 对象触发 onopen 事件，前端立马向后端发送鉴权
- 后端正在接收大模型数据，将数据一边发送给前端，一边存在缓存里，并返回鉴权成功的信息
- 前端接收到鉴权成功信息，创建一个数组专门用于接收数据，并通知后端把所有缓存的数据发送给前端
- 前端接收到缓存数据后，进行去重以及排序，以防抖的频率进行输出

# 🧩 简单的事件订阅器 WebSocket client

SSE 因为是单向的通道，通常情况下已经无法满足 Agent 这种复杂的交互，Agent 中包含有大量的中断、调用工具、反复 AI 调用等逻辑链条，必须要和 LLM 建立一种长久的双向通信通道。所以我们只有唯一的选择：WebSocket。

## 🛠️ 创建 WebSocket client

仔细想一下这个 WebSocket 需要承担什么职责：负责向后端发起 WebSocket 连接，负责中断 WebSocket 连接，负责网络中断时延迟重连。

**所以这个 WebSocket client 必须具有以下几个方法**：

- connect：进行连接，并注册好 WebSocket 的各种 onopen、onmessage、onclose、onerror 事件
- reconnect：清空掉对上一个 WebSocket 的事件监听，重新建立连接
- disconnect：清空掉对上一个 WebSocket 的事件监听
- handleMessage：将 ws 的各种监听发布到总线上
- resume：由于我们必须有以下行为先建立连接后鉴权，但是后端在鉴权的时间里已经向前端发送了很多信息但是前端没有收到，所以必须有一个方法通知后端通过 WebSocket 将信息进行重新发送
- flushResumeBuffer：将后端的信息进行拦截以及去重，最后再进行按顺序排序输出

**由于这几个方法的存在，我们还必须需要这几个变量**：

- status：标识当前这个 client 处于什么状态
- reconnectTimer：重连的 timer
- ws：当前激活的 ws 对象
- sessionEnded：判断后端是否已经终结 Agent 周期
- autoReconnect：判断是否需要重连
- intentionalDisconnect：中止重连标识，禁止以后的所有重连行为
- resumeMode：标识当前正在接收后端的数据
- resumeBuffer：用于接收后端的缓存数据组

### 🔗 链路

-

### connect

```typescript
  doConnect(): {
    this.clearReconnectTimer(); // 清空 timer
    this.setStatus('connecting'); // 标识状态为 connecting

    try {
      const wsUrl = this.buildWsUrl(); // 取得新的 wsurl
      const ws = new WebSocket(wsUrl); // 创建 WebSocket 对象

      ws.onopen = this.handleOpen; // 重写 WebSocket 的各种方法
      ws.onmessage = this.handleMessage;
      ws.onclose = this.handleClose;
      ws.onerror = this.handleError;

      this.ws = ws; // 写入当前激活的对象
    } catch (error) {
      this.setStatus('disconnected'); // 标识状态为 disconnected
      if (this.autoReconnect && !this.sessionEnded) {
        this.scheduleReconnect(); // 进行重连
      }
    }
  }
```

### disconnect

```typescript
  disconnect(): void {
    this.intentionalDisconnect = true; // 禁止重连行为
    this.stopHeartbeat(); //
    this.clearReconnectTimer(); // 清空计时
    this.closeWebSocket(); // 关闭 WebSocket
    if (this.resumeFlushTimer) {
      clearTimeout(this.resumeFlushTimer);
      this.resumeFlushTimer = null;
    }
    this.resumeMode = false;
    this.resumeBuffer = [];
    this.setStatus('disconnected');
    this.emit('disconnected');
  }
```

### reconnect

```typescript
  async reconnect(): Promise<void> {
    this.stopHeartbeat();
    this.clearReconnectTimer();
    this.closeWebSocket();
    if (this.resumeFlushTimer) {
      clearTimeout(this.resumeFlushTimer);
      this.resumeFlushTimer = null;
    }
    this.resumeMode = false;
    this.resumeBuffer = []; // 清空状态
    this.intentionalDisconnect = false; // 标识允许进行重连
    this.sessionEnded = false; // 标识 Agent 未结束
    this.reconnectDelay = INITIAL_RECONNECT_DELAY;
    this.doConnect(); // 开始连接
  }
```

### 📬 message

handleOpen：标识状态为已连接上，并立马发送 Token 给对方。

```typescript
  private handleOpen = (): void => {
    this.reconnectDelay = INITIAL_RECONNECT_DELAY;
    this.setStatus('authenticating'); // 标识为正准备连接
    this.sendMessage({ token: this.token, type: 'auth' }); // 发送 Token 给后端
  };
```

handleMessage：WebSocket 事件中转。

我们需要明白一个需要理解的事实：

```typescript
  private handleMessage = (event: MessageEvent): void => {
    try {
      const message = JSON.parse(event.data as string) as ServerMessage;
      // 解析传回的数据

      switch (message.type) {
        case 'auth_success': {
          // 服务端传回鉴权成功
          this.setStatus('connected'); // 标识连接成功
          this.startHeartbeat(); // 开启心跳重连器
          if (this.resumeOnConnect && !this.lastEventId) { // 如果这次连接是恢复行为并且后端已经结束
            this.resumeMode = true; // 标识启动恢复
            this.resumeBuffer = []; // 清空缓存数组
            this.resumeFlushTimer = setTimeout(() => { // 启动一个计时器，用于判断后端在规定时间内是否有返回数据，防止状态机无限等待后端传回数据
              if (this.resumeMode && this.resumeBuffer.length === 0) {
                this.resumeMode = false;
                this.sessionEnded = true;
                this.emit('session_complete');
                this.disconnect();
              }
            }, RESUME_TIMEOUT);
          }
          this.sendMessage({ lastEventId: this.lastEventId, type: 'resume' }); // 要求后端发送积压数据
          this.emit('connected');

          break;
        }

        case 'auth_failed': {
          this.emit('auth_failed', message.reason);
          this.disconnect();
          break;
        }

        case 'auth_expired': {
          this.emit('auth_expired');
          // 通知外部立马重新鉴权，而不是中断连接
          break;
        }

        case 'heartbeat_ack': {
          this.missedHeartbeats = 0;
          // 后端已经接收到心跳探知
          break;
        }

        case 'agent_event': {
          const agentEvent: AgentStreamEvent = message.event;
          if (message.id) this.lastEventId = message.id;

          if (this.resumeMode) { // 如果此时正在进行缓存输出，则把数据都推入缓存数组，等待泄洪
            this.resumeBuffer.push({ event: agentEvent, id: message.id });
            this.scheduleResumeFlush();
            if (agentEvent.type === 'agent_runtime_end' || agentEvent.type === 'error') {
              this.sessionEnded = true;
              this.flushResumeBuffer();
              this.disconnect();
            }
            break;
          }

          this.emit('agent_event', agentEvent);
          // 正常情况则正常输出
          if (agentEvent.type === 'agent_runtime_end' || agentEvent.type === 'error') {
            // 后端通知结束了则关闭连接
            this.sessionEnded = true;
            this.disconnect();
          }
          break;
        }

        case 'session_complete': {
          this.sessionEnded = true;
          if (this.resumeMode) {
            this.flushResumeBuffer(); // 对话结束，直接执行洗刷数据
          }
          this.emit('session_complete');
          this.disconnect();
          break;
        }
      }
    } catch (error) {
      console.error('[AgentStreamClient] Failed to parse message:', error);
    }
  };
```

### 🧹 数据缓存和洗刷

要维护这样不间断地接收后端的缓存数据，并自感知后端的缓存数据结束，我们需要使用到一个自推迟的机制来实现。

**它的顺序将变成这样**：

- 后端传来数据，使用缓存数组进行接收
- 触发防抖计时器，不断地延迟洗刷数据的到来
- 终于后端在若干时间后没有再触发防抖计时器
- 洗刷数据，标识已经结束了缓存接收
- 数据洗刷完毕，直接循环抛给外部

**防抖计时器**：

```typescript
  private scheduleResumeFlush(): void {
    if (this.resumeFlushTimer) clearTimeout(this.resumeFlushTimer); // 如果已经有计时器，则立马推迟
    this.resumeFlushTimer = setTimeout(() => { // 计时器内部就是中止接收，开始洗刷数据
      this.flushResumeBuffer();
    }, RESUME_FLUSH_DELAY);
  }
```

**数据洗刷器**：

```typescript
  private flushResumeBuffer(): void {
    if (!this.resumeMode) return; // 边界阻止
    this.resumeMode = false; // 标识结束了缓存

    if (this.resumeFlushTimer) { // 结束所有的缓存防抖计时器，边界阻止
      clearTimeout(this.resumeFlushTimer);
      this.resumeFlushTimer = null;
    }

    // 数据去重处理
    const seen = new Set<string>(); // 使用 seen 做 key
    const deduped: AgentStreamEvent[] = []; // 过滤完的数据
    // 开始过滤
    for (const { event, id } of this.resumeBuffer) {
      const key = id || `${event.type}_${event.stepIndex}_${event.timestamp}`;
      if (!seen.has(key)) {
        seen.add(key);
        deduped.push(event);
      }
    }
    this.resumeBuffer = [];

    // 把数据弹出
    for (const event of deduped) {
      this.emit('agent_event', event);
    }
  }
```

## 🚌 创建简洁的事件总线

## 🔄 前端上下文中转
参考
[[agent开发部分/lobehub解析/ai架构/gateway - 前端 - 事件处理器|gateway - 前端 - 事件处理器]]
