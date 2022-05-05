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

