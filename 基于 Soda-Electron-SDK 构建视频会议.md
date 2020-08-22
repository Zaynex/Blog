# 基于 Soda-Electron-SDK 构建视频会议




简介：

本次分享将深入探索有道云会议客户端（windows/pc) 的整体架构以及工作机制（包括Soda-Electron-SDK 构建、音视频传输、媒体播放器等），以及在客户端开发中所遇到的一些问题和经验总结。



第一部分主要介绍 Soda 以及 Soda Electron SDK, 如何构建该 SDK。以及如何获取视频数据。

1. 介绍 Soda 以及 Soda Electron SDK
2. SDK 构建编译流程等。

第二部分介绍 视频数据的格式以及 前端如何渲染该数据——即自制媒体播放器。





通过 SDK 获取到的数据是 **`YUV420p`** 。

YUV 是一种颜色编码方式，与我们熟知的 RGB类似，但它们所代表的含义不同。YUV 中的 “Y” 表示明亮度（Luminance或Luma），也就是灰度值；而“U”和“V” 表示的则是色度（Chrominance或Chroma），作用是描述影像色彩及饱和度，用于指定像素的颜色。

YUV 的编码方式主要用于电视系统以及模拟视频领域，它将亮度信息（Y）与色彩信息（UV）分离，没有UV信息一样可以显示完整的图像，只不过是黑白的，这样的设计很好地解决了彩色电视机与黑白电视的兼容问题。并且，YUV不像RGB那样要求三个独立的视频信号同时传输，所以用YUV方式传送占用极少的频宽。

> 此前已经有文章做过这方面的介绍: [《IVWEB 玩转 WASM 系列-WEBGL YUV 渲染图像实践》](https://juejin.im/post/5de29d7be51d455f9b335efa)。



现在我们面临的问题是，如何将 YUV 数据渲染出来。

这里我们采用的是 Yuv-Buffer + YUV Canvas | WebGl 渲染图像。



第三部分介绍 客户端整体架构和框架选型（为何选用 Electron 等）



第四部分介绍实践中的一些处理和优化，Crash 监控



第五部分，后续优化以及如何开放该 SDK 给更多的开发者使用。

目前 sdk 的一些问题和可被拓展实现的一些功能。

1. 如何分享指定的应用屏幕数据，而非屏幕分享。
2. 如何做到远程控制。
3. 如何实现屏幕上标注。





###  what is yuv and how difference from RGB ?




### YUV render on canvas(webgl)
- yuv-buffer
- yuv-canvas

## 性能优化
1. 降低帧率
2. 降低分辨率
3. web worker


## 渲染进程间数据同步

Electron 中渲染进程间数据是不同步的。

虽然在我们的页面，是通过加载同一份 JavaScript 代码运行。但是 渲染进程 A 中执行时保存的状态并不能在渲染进程 B 中同步。

Electron 提供一个 全局变量用以共享进程间的数据。

即 global 对象。但它不具备事件机制，没有通信功能。即使在 global 上设置了新的数据，其他使用到该数据的进程无法感知到。

我们需要一种消息机制来告知进程有数据变更。即 IPC。
通过 IPC 调用，类似发布订阅模式。

所以，对于会议期间的用户列表在渲染进程间更新的模式方式是：

1. 渲染进程A 收到 Soda 新用户加入，写入数据，并通知主进程
2. 主进程收到通知后，转发给另外的渲染进程B
3. 渲染进程B 收到后，读取数据并写入

在真正的产品中，渲染进程写入到 indexedDB，待 DB 更新后告知其他渲染进程从 indexedDB 获取新的数据。

这里可能会有人觉得这样每次渲染进程都是从数据库读取完整的数据，而不是增量更新，会不会有执行时的性能问题？

而其实 indexedDB 的数据读写是异步的，(而且支持在 web worker 中使用)并不阻塞线程。

如果要进行增量更新的话，模式就会成为命令模式，即 （type, data) 的组合值，这样虽然做到了精细化，性能最优，但却增加了编码的复杂度。（需要根据不同的类型匹配对应的处理）。

在项目前期我们选择了对性能影响不那么大且快速的方案。

## 状态管理

关于状态管理，起初采用的是 unstated。但是它对 hooks 支持并不友好。后面我们采用的是 gitbook 自定义的 unstated. 它能友好得在 hooks 中使用。



## 性能优化

### 消息优化

场景：每次当用户加入时，我们都会收到 useradd 指令。假设此时会议室里已经有 200个用户，那么新用户加入后，会收到 200 个通知。而每次通知都会触发 UI 渲染以及其他操作。

而由于 JavaScript 执行又是单线程的。我们会被卡在这200个消息事件的处理上。



我们可以加缓存，当消息来到时，我们先缓存起来，不急着触发 UI 更新，待满足合适的条件（比如在 300ms 内或者缓存的消息数打到30个时）我们批量更新 UI 数据。



这里，我们使用 rxjs 来帮助我们做上述优化



#### BufferCount 

![img](https://cn.rx.js.org/img/bufferCount.png)

Collect emitted values until provided number is fulfilled, emit as array. 



```javascript
import { bufferCount } from 'rxjs/operators';
const subject = new Subject();

event.on(EVENT.useradd, (user) => subject.next(user))

const buffferTen = subject.pipe(bufferCount(10))
const subscribe = buffferTen.subscribe(val =>
  console.log('Buffered Values:', val)
  dispatch.addUsers(val);// update state
);
```

这样的话，我们就能做到缓存。然而，仅此优化还不够。假设，只有 8个用户加入没有达到缓存上限是不会收到 Subscribe 的。



我们需要使用 BufferTime

![img](https://cn.rx.js.org/img/bufferTime.png)

```
import { bufferTime } from 'rxjs/operators';
const subject = new Subject();

event.on(EVENT.useradd, (user) => subject.next(user))
// 每隔 2s 或者当缓冲达到10个时
const buffferTen = subject.pipe(bufferTime(2000, null, 10))
const subscribe = buffferTen.subscribe(val =>
  console.log('Buffered Values:', val)
  dispatch.addUsers(val);// update state
);
```

它能满足我们的实时性和性能要求。

## 如何做到工业级水准

1. javascript 是一门动态运行时的脚本语言。

   1.1.  this 的指向，动态时才能确定。

2. Crash 监控

3. 异常提示

4. 代码可拓展/可维护

## Technology

View: React,  @material-ui, styled-components

Statemangment: <del>unstated</del>  @ice/store , native hooks, store, electron-store

Library: Rxjs , lodash, request, electron-better-ipc,electron-updater,idb-keyval





## Reference

1. https://juejin.im/post/5de29d7be51d455f9b335efa
2. 