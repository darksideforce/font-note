---
aliases:
  - SSE 的实现与原理
tags:
  - 后端
  - 网络
  - SSE
status: 待完善
created: 2026-07-07
---

# 概述 🧭

在了解 SSE 之前，我们需要先了解一些关于 SSE 的网络层知识。

**Stream 对象**
为了解决大数据传输问题而产生的对象，通过约定的方式包含一些方法供外部调用。

**迭代器**
一种按约定逐个取值的协议。异步迭代器的 `next()` 会返回 Promise，`await` 只是等待这次取值完成，不会阻塞整个进程。

最小链路
```text
OpenAI 服务端返回 HTTP 流式响应
  ↓
fetch 得到 Response.body: ReadableStream<Uint8Array>
  ↓
OpenAI SDK 读取 response.body
  ↓
按 SSE 的 data: ...\n\n 边界解析
  ↓
JSON.parse 成 JS chunk
  ↓
SDK 暴露为 AsyncIterable
  ↓
Lobe 的 chatStreamable 只是 for await 后原样 yield
  ↓
convertIterableToStream 再把 AsyncIterable 接到 ReadableStream.pull
```

## Stream 对象 🔁

它是一个“可被逐块读取的数据源对象”，允许我们每次只读取一段数据。

Node.js 和浏览器内置的 Stream 本质上是一种对象封装，Stream 对象包含这三种：

- **`ReadableStream`**
- **`TransformStream`**
- **`WritableStream`**

以 `ReadableStream` 为例，创建时可以提供 `start`、`pull`、`cancel` 这三种底层数据源钩子，用来描述数据如何开始、如何按需拉取、如何取消。

Stream 对象本质上是一种带有**背压（Backpressure）控制的生产者-消费者队列**，也是一个管道对象。内部有一个 `controller` 给**生产者**用来推数据（`enqueue`），外部暴露了一个 `reader` 给**消费者**用来拉数据（`read`）。只要一有新数据入队，挂在消费者那头的 `Promise` 就会被解析（resolve），从而触发外部的回调逻辑。

Streamable 对象本质上是一种双方进行协调的结果。

`pull`、`start`、`cancel` 这三种方法不是让业务消费者直接调用的公开方法，而是交给 Stream 运行时在合适时机调用的钩子：

- `start`：ReadableStream 创建并初始化底层数据源时执行。
- `pull`：消费者调用 `read()` 后，Stream 运行时发现需要更多数据时执行。
- `cancel`：消费者取消读取或流被关闭时执行。

你可能看完会想：这样一个对象是如何实现与大模型侧的交互呢？谁来决定拉取数据？又是谁返回数据？两者之间是如何联系在一起的？

我们先确定两个概念：生产者和消费者。

- 如何交互：底层的 Web 引擎会将这个 Streamable 对象交给消费者，消费者调用的是 `reader.read()`；当运行时需要更多数据时，才会触发底层的 `pull` 钩子。但是在源头，它并不是 Streamable，而是一个我们下面会提到的迭代器协议。
- 谁在返回数据：一般是请求层。我们可以通过特殊的请求方式封装不同的请求，例如 `HTTP chunked`、`HTTP/2 DATA frames`、`TCP stream`。
- 两者之间如何联系在一起：消费者通过调用 Streamable 对象的 `read` 触发读取流程；如果内部缓冲不够，运行时会调用 `pull`，`pull` 再推进上游数据源，例如调用迭代器的 `next()`。

链路：

```text
SDK 将 HTTP 流式响应包装成 AsyncIterable，迭代器的 next 方法会推进本地读取和解析流程
  ↓
创建 Streamable 对象，pull 方法会指向迭代器的 next 方法
  ↓
经过管道等一系列方法，最终形成完整的 Streamable 对象，交给消费者
  ↓
消费者调用 read 等方法消费 Streamable，最终连接到顶层
```


## 迭代器 🧩

接下来要讲一下为什么 Streamable 对象可以成为 SSE 中连接消费者和生产者的核心，答案就是迭代器。

迭代器本质上是一个可以逐个输出内容的对象：

```ts
const iterator = {
  next() {
    return {
      value: '当前值',
      done: false,
    };
  },
};
```

一个对象只要有 `next()` 方法，并且 `next()` 每次返回 `{ value: 某个值, done: 是否结束 }`，那这就是一个迭代器。

进一步，我们可以让这个 `next` 方法变成异步的。异步的 `next` 返回的是一个 `Promise` 对象：

```ts
const asyncIterator = {
  async next() {
    return {
      value: '你',
      done: false,
    };
  },
};
```

这样的模型在 SSE 通信时会相当好用。真实网络接口不会直接返回迭代器，但 SDK 可以把 HTTP 流式响应包装成 AsyncIterable。每次调用 `next()` 时，它会推进本地读取和解析流程；如果本地还没有完整事件，就等待网络数据继续到达。远端服务端不是被每次 `next()` 直接命令生成下一帧，而是通过底层缓冲和背压间接协调。

我们还需要知道一个东西：可迭代对象。

## 真实的网络层 🌐


当然，真实的网络层并不会直接返回一个迭代器对象，这肯定是无法做到的。那我们需要如何把这一步封装成可迭代对象呢？

```text
OpenAI 服务端持续写 HTTP response
  ↓
网络 / TCP / HTTP 层接收数据
  ↓
浏览器或 Node fetch 把数据放进 response.body 的内部缓冲区
  ↓
SDK 调 response.body 的 reader.read()
  ↓
读出 Uint8Array 字节块
  ↓
SDK 解码文本、解析 SSE、JSON.parse
  ↓
yield 一个 JS chunk
  ↓
调用 iterator.next() / for await 拿到这个 chunk
```

伪代码：

1. SDK 发起请求，拿到 HTTP Response。

```ts
async function openaiSdkCreateStream() {
  const response = await fetch('https://api.openai.com/v1/chat/completions', {
    method: 'POST',
    body: JSON.stringify({
      stream: true,
      messages: [],
    }),
  });

  // response.body 是 ReadableStream<Uint8Array>
  return createSdkStreamFromResponse(response);
}
```

2. 通过闭包拿到 `response` 的指向，在 `createSdkStreamFromResponse` 中对读取方法进行一层包装，返回一个可迭代对象。

```ts
function createSdkStreamFromResponse(response: Response): AsyncIterable<any> {
  return {
    async *[Symbol.asyncIterator]() {
      const reader = response.body!.getReader();
      const decoder = new TextDecoder();

      let buffer = '';

      while (true) {
        const { value, done } = await reader.read();

        if (done) break;

        // Uint8Array -> string
        buffer += decoder.decode(value, { stream: true });

        // SSE 用空行分隔事件
        const events = buffer.split('\n\n');
        buffer = events.pop() ?? '';

        for (const event of events) {
          const line = event
            .split('\n')
            .find((line) => line.startsWith('data: '));

          if (!line) continue;

          const data = line.slice('data: '.length);

          if (data === '[DONE]') return;

          // 这里 yield 出 SDK 解析好的 chunk
          yield JSON.parse(data);
        }
      }
    },
  };
}
```

3. 再通过 async generator 原样转发这个可迭代对象。

```ts
const chatStreamable = async function* <T>(stream: AsyncIterable<T>) {
  for await (const response of stream) {
    yield response;
  }
};
```

本质上，`chatStreamable` 每次被外部推进时，都会继续消费传入的 AsyncIterable。它会进入 `createSdkStreamFromResponse` 创建的异步迭代流程，但一次 `next()` 不一定等于一次 `reader.read()`：可能读 0 次、1 次或多次，直到解析出一个完整 chunk 后才 `yield`。

## 接口如何等待消费者的调用

还有最后一个问题。我也是困扰了许久。我们之前已经提到了。消费者的消费需要通知到生产者，而生产者又是如何做到等待消费者的这次消费结束的呢？
答案是缓冲区和背压。在源头的协议交互层，远端服务端并不会等待本地消费者调用 `next()` 以后才生成下一帧，也不是靠某种固定间隔输出。
那它怎么做到的和消费者进行协调的呢？
答案是背压：
```
  -> 客户端本地缓冲区逐渐堆积
  -> 操作系统 TCP 接收缓冲区逐渐堆积
  -> TCP 窗口变小
  -> 服务端写 socket 变慢或阻塞
  -> 服务端发送节奏被压住
```
所以并不是直觉上看到的，消费者消费完毕后直接触发远端 `next()`，而是
```
消费者读得慢
  -> 网络/运行时缓冲区满
  -> 服务端写不动
  -> 上游被迫慢下来
```
真正影响本地消费的是 `response.body` 这一步的 `read()`：SSE 接口会持续返回新数据，客户端运行时先把数据放进缓冲区；如果缓冲区没满，服务端可以继续写；如果消费者长期不读导致缓冲区满，背压才会让上游写入变慢或阻塞。`read()` 是单向消费过程，读过的数据不能回溯，同一个 stream 通常也不能被重复读取。


