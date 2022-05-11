### 问题

本文会总结出使用service worker可能会遇到的问题及解决办法。

### service-worker更新和缓存
问题描述：<br>
* service-worker.js也会受http的缓存策略控制
* 如果新的worker未被成功下载，或者解析错误，或者在运行时出错，或者在安装阶段不成功，新的worker会被丢弃，旧的会被保留
* 一旦新的worker被成功安装，更新的worker会进入等待状态，新的worker会等待旧的worker下线才会激活，新的worker和旧的会并存
* self.skipWaiting()会强制跳过等待状态，直接让新的worker在安装后进入激活状态，这样可能会有缓存问题
* 浏览器会 diff 当前打开页面的 service-worker.js，并判断是否更新，如果 diff 结果为更新，则重新安装最新的 service-wroker.js，并且全量更新缓存
* 任何静态资源包括 service-worker.js 都会被 HTTP 缓存
* 服务器对某个资源进行 no-cache 设置可以避免 HTTP 缓存


解决办法：<br>
针对上述的情况，service-worker的更新就是必须解决的问题。
1. 在服务器端配置service-worker的header，Cache-control: no-cache，使其不被缓存
2. 前端进行service-worker的版本控制，每次注册都添加版本号进行改写

```javascript
const cacheVersion =
  process.env.NODE_ENV === "production"
    ? process.env.CACHE_VERSION
    : Date.now();

if ("serviceWorker" in navigator) {
  register(
    `${process.env.BASE_URL}service-worker.js?cacheVersion=${cacheVersion}`,
    {
      ready() {
        console.log(
          "service worker正在从缓存中为app提供服务.\n" +
            "查看更多, 访问 https://goo.gl/AFskqB"
        );
      },
    }
  );
}
```

### service-worker激活
问题描述：<br>
由于浏览器的内部实现原理，当页面切换或者自身刷新时，浏览器是等到新的页面完成渲染之后再销毁旧的页面。这表示新旧两个页面中间有共同存在的交叉时间，因此简单的切换页面或者刷新是不能使得 service worker 进行更新的。

解决方案：<br>
既然service-worker的激活无法通过刷新解决，那么还有个skipWaiting可以用。<br>
但是最好不要直接skipWaiting(跳过等待阶段)， 推荐的做法应该是在浏览器发现更新后，给用户弹出提示。然后用户点击重新加载时，一方面刷新页面 (location.reload())，一方面让新的 SW 接管页面 (skipWaiting)。

具体流程：<br>
* 在注册service-worker时就监听sw的更新状况
* 如果有更新，并且安装完成后，就发送自定义事件sw.update
* 自定义事件被触发，显示更新按钮
* 用户点击更新按钮触发更新

```javascript




```

### 当service worker安装失败怎么处理
问题描述：<br>
当service-worker新版本的更新出现问题，如何保证用户看到的版本是最新的。
解决办法：<br>
卸载当前的sw，用线上的文件，并且不再安装当前错误版本的。
