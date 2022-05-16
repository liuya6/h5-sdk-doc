### 功能
```
  -打包开启gzip压缩
  -打包分离公共代码
  -打包压缩图片
  -组件按需加载 
  -全局scss变量
  -移动端布局（pxtorem）
  -axios请求API封装
  -常用工具函数（节流，防抖，深克隆等）
  -service worker离线缓存
  -后续还会添加更多功能
```

### 打包压缩gzip
> 作用：压缩代码体积，提高加载速度。<br>

首先安装 compression-webpack-plugin

```
yarn add compression-webpack-plugin -D (开发依赖)
```  
  然后在vue.config.js文件中添加以下代码开启
```javascript
const CompressionWebpackPlugin = require("compression-webpack-plugin");
module.exports = defineConfig({
  configureWebpack: (config) => {
    if (isProduction) {
      config.plugins.push(new CompressionWebpackPlugin()); //开启gzip
    }
  },
});
```
  随后在命令行运行 
```
yarn build
```
  可以看到打包已经产出了gzip格式的文件，比原文件体积压缩了有百分之60以上。
![Chrome 下运行 Service Worker 示例的结果](./img/gzip.jpg)


### 打包分离公共代码
> 作用：因为http请求是并发的（可以同时请求多个），把代码分离之后，速度会得到巨大提升。

在vue.config.js文件中添加一下代码

```javascript
module.exports = defineConfig({
  configureWebpack: (config) => {
    config.optimization = {
      splitChunks: {
        cacheGroups: {
          vendors1: {
            name: "chunk-vendors1",
            test: /[\\/]node_modules[\\/](vue|vant)/, // 把vue,vant打包在一起
            priority: 20,
            chunks: "initial",
          },
          vendors2: {
            name: "chunk-vendors2",
            test: /[\\/]node_modules[\\/]/, // 余下的三方库打包在一起
            priority: 10,
            chunks: "initial",
          },
        },
      },
    };
  },
});
```
查看效果
![Chrome 下运行 Service Worker 示例的结果](./img/separate.jpg)


### 打包压缩图片
首先安装 image-webpack-loader
```
yarn add compression-webpack-plugin -D (开发依赖)

```  
如果遇到打包失败（网速原因东西下载不完整导致）就使用cnpm安装
```
cnpm i image-webpack-loader -D
```
然后在vue.config.js文件中添加以下代码开启
```javascript
module.exports = defineConfig({
  chainWebpack: (config) => {
    config.module
      .rule("images")
      .test(/\.(png|jpe?g|gif)(\?.*)?$/)
      .use("image-webpack-loader")
      .loader("image-webpack-loader")
      .options({ bypassOnDebug: true })
      .end();
  },
});
```
随后在命令行运行
```
yarn build
```
可以看到打包产出了图片文件体积，比原文件体积压缩了有百分之70以上。<br/>
`压缩前`
![Chrome 下运行 Service Worker 示例的结果](./img/img1.jpg)
`压缩后`
![Chrome 下运行 Service Worker 示例的结果](./img/img2.png)

### 组件按需加载

> 作用：为了防止首页加载文件过大，导致长时间白屏，影响用户体验

```
{
    path: "/rank",
    name: "ranking",
    component: () => import("../views/Home/components/Ranking.vue"), // 按需加载引入组件
    meta: {
      keepAlive: true,
    },
}

```

### 全局scss变量
> 作用：定义好全局主题色，加快开发速度

首先安装style-resources-loader

```
yarn add style-resources-loader -D  // 开发依赖

```

然后在vue.config.js中添入一下代码

```
module.exports = defineConfig({
  pluginOptions: {
    "style-resources-loader": {
      preProcessor: "scss",
      patterns: [path.resolve(__dirname, `./src/style/theme/index.scss`)],
    },
  },
});

```

使用方法：<br>
index.scss内容
```scss
$theme: #c44f41;

```
其他vue文件中，可以直接使用index.scss定义的内容，不需要引入scss文件，如下所示

```scss
<style lang="scss" scoped>
.test {
  color: $theme;    
}
</style>
```

### 移动端布局（pxtorem）
> 作用: 可直接根据设计稿尺寸来写px单位，插件会自动把px转换为rem来实现移动端布局。<br>

首先安装postcss-pxtorem

```
yarn add postcss-pxtorem -D (开发依赖)

```

然后在postcss.config.js中输入以下代码

```js
module.exports = {
  plugins: {
    "postcss-pxtorem": {
      rootValue: 37.5,  // 375的设计稿是37.5 同理750设计稿为75
      propList: ["*"],
    },
  },
};
```

### axios封装
位于`src/api/http.ts`文件中。

> request拦截，配置token等请求头参数。<br>

```typescript
instance.interceptors.request.use(
  function (config: AxiosRequestConfig) {
    if (!config.cancelToken) {
      const source = axios.CancelToken.source();
      config.cancelToken = source.token;
      CancelTokenSourceMap.set(config.url, source);
    }
    config.headers = Object.assign(
      {
        "os-type": 1,
      },
      config.headers
    );
    return config;
  },
  function (error) {
    return Promise.reject(error);
  }
);

```

> response返回接口拦截，对接口状态统一拦截处理。<br>

```typescript
instance.interceptors.response.use(
  function (response: AxiosResponse<any>) {
    // 删除cancel token
    CancelTokenSourceMap.delete(response.config.url);
    const responseData = response.data;
    // 后台业务错误
    if (responseData.errorCode) {
      if (response.request.responseURL.indexOf("single-newest-result") === -1) {
        dealApiError(responseData);
      }
      return Promise.reject(responseData);
    }
    return response.data;
  },
  function (error: AxiosError<ResponseError>) {
    if (error && error.toString && error.toString() === "Cancel") {
      return Promise.reject(error);
    }
    return Promise.reject(error);
  }
);

```

使用<br>

```typescript
import { http } from "../http";

export function getHomePage() {
  return http.get("api/xxx");
}

```

### 常用工具函数

常用工具函数，可写在`src/util`文件夹内，需要的可以自行添加。<br>

目前已经有了几个常用的函数。

例如：
```typescript
// 防抖
const throttle = (fn: () => void, delay = 1500): (() => void) => {
  let flag = true;
  let timer = setTimeout(() => {}, delay);
  return function () {
    if (flag) {
      flag = false;
      clearTimeout(timer);
      timer = setTimeout(() => {
        fn();
        flag = true;
      }, delay);
    }
  };
};
// 节流
const debounce = (fn: () => void, delay = 1500): (() => void) => {
  let timer = setTimeout(() => {}, delay);
  return function () {
    clearTimeout(timer);
    timer = setTimeout(() => {
      fn();
    }, delay);
  };
};
// 生成[n,m]的随机整数
function randomNum(minNum: number, maxNum: number) {
  switch (arguments.length) {
    case 1:
      return Math.floor(Math.random() * minNum + 1);
    case 2:
      return Math.floor(Math.random() * (maxNum - minNum + 1) + minNum);
    default:
      return 0;
  }
}

```

### service worker离线缓存
> 作用： 提供离线或者弱网环境的用户体验。 <br>
 
首先安装 register-service-worker插件，帮助注册service-worker
```
yarn add register-service-worker -s (项目依赖)

```

然后 在src文件夹下创建registerServiceWorker.ts,内容如下

> 这段代码主要是为了注册使用service-worker

```typescript
import { register, unregister } from "register-service-worker";

import { Dialog, Notify } from "vant";

const cacheVersion = "v1.1.1";

if ("serviceWorker" in navigator) {
  // unregister();
  register(
    `${process.env.BASE_URL}service-worker.js?cacheVersion=${cacheVersion}`,
    {
      ready() {
        console.log(
          "service worker正在从缓存中为app提供服务.\n" +
            "查看更多, 访问 https://goo.gl/AFskqB"
        );
      },
      registered(registration) {
        if (registration.waiting) {
          return;
        }
        Notify({
          type: "success",
          message: "service worker 已注册",
        });
      },
      cached() {
        Notify({
          type: "success",
          message: "内容已缓存以供离线使用",
        });
      },
      updatefound() {
        // Notify({
        //   type: "success",
        //   message: "service worker 正在下载新内容",
        // });
        console.log("service worker 正在下载新内容");
      },
      updated(registration) {
        const worker = registration.waiting;
        if (worker) {
          const { cacheVersion, oldCacheVersion } = getNewAndOldVersion(worker);
          // 要更新的sw版本和正在使用的sw版本一致就return出去
          if (cacheVersion === oldCacheVersion) return;

          Dialog.confirm({
            title: "提示",
            message: "有新内容可用；请刷新。",
          }).then(() => {
            navigator.serviceWorker
              .getRegistration()
              .then(() => {
                // 如果有新版本，就先清空缓存
                clearCache();
                // 直接启用新版本sw
                skipWaiting(registration);
                window.localStorage.setItem("cacheVersion", cacheVersion);
              })
              .then(() => {
                window.location.reload();
              });
          });
        }
      },
      offline() {
        Notify({
          type: "danger",
          message: "未找到互联网连接。应用程序正在离线模式下运行。",
        });
      },
      error(error) {
        Notify({
          type: "danger",
          message: `Service Worker 注册期间的错误：${error}`,
        });
        unregister(); // 注册期间失败直接卸载sw
        console.log(error, "error");
      },
    }
  );
}

function skipWaiting(registration: any) {
  const worker = registration.waiting;
  if (!worker) {
    return Promise.resolve();
  }
  // 这里是参考vue-press的写法
  // 利用MessageChannel返回一个promise
  return new Promise((resolve, reject) => {
    const channel = new MessageChannel();
    channel.port1.onmessage = (event) => {
      if (event.data.error) {
        reject(event.data.error);
      } else {
        resolve(event.data);
      }
    };
    worker.postMessage({ type: "skip-waiting" }, [channel.port2]);
  });
}

function getNewAndOldVersion(worker: any) {
  const url = new URL(worker?.scriptURL);
  const params = new URLSearchParams(url.search);
  const cacheVersion = params.get("cacheVersion") || "";
  const oldCacheVersion = window.localStorage.getItem("cacheVersion");

  return {
    cacheVersion,
    oldCacheVersion,
  };
}

function clearCache() {
  caches.keys().then(function (cacheList) {
    return Promise.all(
      cacheList.map(function (cacheName) {
        return caches.delete(cacheName);
      })
    );
  });
}

```

最后在`public`文件夹下创建service-worker.js，内容如下：

> service-worker主要代码，用于控制http请求缓存的处理。

```typescript
importScripts(
  "https://cdn.bootcdn.net/ajax/libs/workbox-sw/6.5.3/workbox-sw.min.js"
);

if (workbox) {
  console.log("workbox加载成功。");
  self.addEventListener("message", (event) => {
    const replyPort = event.ports[0];
    const message = event.data;
    if (replyPort && message && message.type === "skip-waiting") {
      event.waitUntil(
        self
          .skipWaiting()
          .then(() => {
            workbox.core.clientsClaim();
            replyPort.postMessage({ error: null });
          })
          .catch((error) => replyPort.postMessage({ error }))
      );
    }
  });

  // 删除所有log
  workbox.setConfig({ debug: false });

  // Workbox 加载完成

  workbox.core.setCacheNameDetails({
    prefix: "h5-sdk",
    suffix: "v1",
    precache: "precache",
    runtime: "runtime",
  });

  // 删除过期缓存
  workbox.precaching.cleanupOutdatedCaches();

  // 预缓存 index.html
  workbox.precaching.precacheAndRoute([
    {
      url: "/index.html",
      revision: "1.0.1",
    },
  ]);

  // js css的缓存策略
  workbox.routing.registerRoute(
    /.*.(?:js|css|json|ico)/,
    new workbox.strategies.StaleWhileRevalidate({
      cacheName: "js-css-json-ico-caches",
      plugins: [
        // 需要缓存的状态筛选
        new workbox.cacheableResponse.CacheableResponsePlugin({
          statuses: [0, 200, 304],
        }),
        // 缓存时间
        new workbox.expiration.ExpirationPlugin({
          maxEntries: 20,
          maxAgeSeconds: 60 * 60,
        }),
      ],
    })
  );

  // api接口的缓存策略
  workbox.routing.registerRoute(
    /\/api/,
    new workbox.strategies.NetworkFirst({
      // 可能存在一些网络请求，他们花费的时间很长，那么通过一些配置，让任何在超时内无法响应的网络请求都强制回退到缓存获取。
      networkTimeoutSeconds: 10,
      cacheName: "api-caches",
      plugins: [
        // 需要缓存的状态筛选
        new workbox.cacheableResponse.CacheableResponsePlugin({
          statuses: [0, 200],
        }),
        // 缓存时间
        new workbox.expiration.ExpirationPlugin({
          // 缓存最多50个请求
          maxEntries: 50,
          // 缓存一小时
          maxAgeSeconds: 60 * 60,
        }),
      ],
    })
  );

  // 图片的缓存策略
  workbox.routing.registerRoute(
    /\.(jpe?g|png|svg)/,
    new workbox.strategies.CacheFirst({
      cacheName: "image-cache",
      fetchOptions: {
        mode: "cors",
      },
      plugins: [
        new workbox.expiration.ExpirationPlugin({
          // 对图片资源缓存 1 星期
          maxAgeSeconds: 7 * 24 * 60 * 60,
          // 匹配该策略的图片最多缓存 10 张
          // maxEntries: 10,
        }),
      ],
    })
  );
}

```

本人有写了一篇service-worker文档,如果之前未接触过，可前往查阅[service-worker教程](https://service-worker-doc.vercel.app/)。

