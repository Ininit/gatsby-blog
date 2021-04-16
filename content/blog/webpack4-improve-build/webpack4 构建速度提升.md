---
title: webpack4 构建速度提升
thumbnail: ./assets/webpack.png
date: 2020-04-28 20:56:19
tags: ['webpack']
---

#### #0x01 性能瓶颈分析

在这里我使用的是`speed-measure-webpack-plugin` 插件进行构件速度分析

```javascript
// https://www.npmjs.com/package/speed-measure-webpack-plugin
// speed-measure-webpack-plugin 简单使用方法
yarn add -D speed-measure-webpack-plugin

// usage
const SpeedMeasurePlugin = require("speed-measure-webpack-plugin")

const smp = new SpeedMeasurePlugin()

const webpackconfig = smp.wrap({
	plugins: [
		...
	]
})
```

获得构建速度数据

1. 总过耗时 240s
2. Plugins 耗时最严重是`ProgressPlugin` 耗时 10s
3. Loaders 耗时严重的有`url-loader` 55s、`vue-loader` 80s、`babel-loader` 40s
4. 基于 `development`模式，每次 rebuild 需要 3s 以上

#### #0x02 针对上面做数据做优化

##### 1. [优化 DevTool](https://webpack.js.org/configuration/devtool/#root)

1. Production 模式
   1. 如果需要隐藏直接 none 模式即可
   2. 如果需要搭配 sentry 这类 bug 追踪服务推荐使用 `hidden-source-map`
2. Development 模式
   1. eval
   2. eval-source-map
   3. eval-cheap-source-map
   4. eval-cheap-module-source-map ☑️

##### 2. 优化 loader 解析时间

1. thread-loader

   将这个 loader 放在其他 loader 即可

   ```javascript
   // thread-loader 预热
   const jsWorkerPool = {
     // options

     // 产生的 worker 的数量，默认是 (cpu 核心数 - 1)
     // 当 require('os').cpus() 是 undefined 时，则为 1
     workers: 2,

     // 闲置时定时删除 worker 进程
     // 默认为 500ms
     // 可以设置为无穷大， 这样在监视模式(--watch)下可以保持 worker 持续存在
     poolTimeout: 2000
   };

   const vueWorkerPool = {
     // 一个 worker 进程中并行执行工作的数量
     // 默认为 20
     workerParallelJobs: 2,
     poolTimeout: 2000
   };

   threadLoader.warmup(jsWorkerPool, ['babel-loader']);
   threadLoader.warmup(vueWorkerPool, ['vue-loader']);

   const webpackConfig = {
     ...
     module: {
       rules: [
         {
           test: /\.(js|jsx)/,
           exclude: /node_modules/,
           use: [
             {
               loader: 'thread-loader',
               options: jsWorkerPool
             },
             'babel-loader?cacheDirectory'
           ]
         },
         // vue
         {
           test: /\.vue$/,
           use: [
             {
               loader: 'thread-loader',
               options: vueWorkerPool
             },
             'vue-loader'
           ]
         }
       ]
     }
     ...
   }
   ```

##### 3. 利用缓存

使用 webpack 缓存的方法有几种，例如使用 `cache-loader`，`HardSourceWebpackPlugin` 或 `babel-loader` 的 `cacheDirectory` 标志。 所有这些缓存方法都有启动的开销。 重新运行期间在本地节省的时间很大，但是初始（冷）运行实际上会更慢

1. cache-loader 与 thread-loader 使用方式一样，仅在开销大的 loader 之前添加该 loader, 将结果缓存到磁盘里面， 显著提升二次构建速度（我的磁盘太烂没有用）

2. [HardSourceWebpackPlugin ](https://github.com/mzgoddard/hard-source-webpack-plugin)

   ```javascript
   new HardSourceWebpackPlugin({
         // cacheDirectory是在高速缓存写入。默认情况下，将缓存存储在node_modules下的目录中
         // 'node_modules/.cache/hard-source/[confighash]'
         cacheDirectory: path.join(__dirname, './lib/.cache/hard-source/[confighash]'),
         // configHash在启动webpack实例时转换webpack配置，
         // 并用于cacheDirectory为不同的webpack配置构建不同的缓存
         configHash: function(webpackConfig) {
           // node-object-hash on npm can be used to build this.
           return require('node-object-hash')({sort: false}).hash(webpackConfig);
         },
         // 当加载器、插件、其他构建时脚本或其他动态依赖项发生更改时，
         // hard-source需要替换缓存以确保输出正确。
         // environmentHash被用来确定这一点。如果散列与先前的构建不同，则将使用新的缓存
         environmentHash: {
           root: process.cwd(),
           directories: [],
           files: ['package-lock.json', 'yarn.lock'],
         },
         // An object. 控制来源
         info: {
           // 'none' or 'test'.
           mode: 'none',
           // 'debug', 'log', 'info', 'warn', or 'error'.
           level: 'debug',
         },
         // Clean up large, old caches automatically.
         cachePrune: {
           // Caches younger than `maxAge` are not considered for deletion. They must
           // be at least this (default: 2 days) old in milliseconds.
           maxAge: 2 * 24 * 60 * 60 * 1000,
           // All caches together must be larger than `sizeThreshold` before any
           // caches will be deleted. Together they must be at least this
           // (default: 50 MB) big in bytes.
           sizeThreshold: 50 * 1024 * 1024
         },
       }),
       new HardSourceWebpackPlugin.ExcludeModulePlugin([
         {
           test: /.*\.DS_Store/
         }
       ]),
   ```

##### 4. DllPlugin 加速

将不经常更换的依赖包进行拆解打包处理，提升构建速度。这里我采用 autodll-webpack-plugin

```javascript
// https://github.com/asfktz/autodll-webpack-plugin/tree/master/experiments/inherit
const AutoDllPlugin = require('autodll-webpack-plugin')
// dll 打包注入
new AutoDllPlugin({
  inject: true, // will inject the DLL bundles to index.html
  filename: '[name].js',
  entry: {
    vendor: [
      'vue',
      'vuex',
      'vue-router'
    ],
    util: [
      'vue-cropper',
      'viser-vue',
      'vue-lazyload',
      'echarts',
      'moment',
    ],
    atdv: [
      'ant-design-vue'
    ]
  }
}),

```

#### #0x03 结果

1. 总过耗时 43s
2. Plugins 耗时最严重是`ProgressPlugin` 耗时 10s
3. Loaders 耗时严重的有`url-loader` 18s、`vue-loader` 55s、`babel-loader` 46s
4. 基于 `development`模式，每次 rebuild 需要 1.5s 左右

#### #0x04 参考链接

1. https://github.com/asfktz/autodll-webpack-plugin/tree/master/experiments/inherit
2. https://github.com/sisterAn/blog/issues/63
