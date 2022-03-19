## 常用配置项

```javascript
module.exports = {
  // 1. 入口文件
  entry: './src/main.js', // string | object | array

  // 2. 出口文件
  output: {
    path: path.resolve(__dirname, 'dist'), // 出口文件的绝对路径
    filename: [name].js, // 出口文件的名字
    publicPath: 'https://cdn.com', // 引用出口文件的路径前缀
    chunkFilename: [chunkhash].js, // 分块文件名字
  },

  // 3. 模块配置(配置loader等)
  module: {
    rules: [ // 加载loader的规则
      {
        test: /\.js?$/, // 匹配的文件
        include: [path.resolve(__dirname, 'src')], // 包含
        exclude: /node_modules/, // 排除
        use: ['xxx', {loader: 'xxx', options: {...}}], // 应用多个loader
        loader: 'babel-loader', // loader
        query/options: {} // loader的配置项
      }
    ]
  },

  // 4. resolve模块解析请求，其实就是将js的模块查找实现一遍
  resolve: {
    modules: ['node_modules'], // 用于查找模块的目录
    extendsions: ['.js'], // 使用的扩展名
    alias: {'vue$': 'vue/dist/vue.esm.js'}, // 设置别名
  },

  // 5. devtool配置source map
  devtool: 'source-map',

  // 6. 不打包的模块
  externals: ['react'],

  // 7. 开发配置
  devServer: {
      port: 8080, // webpack-dev-server的端口
      host: 'localhost', // 服务的host
      allowedHosts: ['xx.com'], // 允许访问服务的host
    proxy: {'/api': 'http://localhost:3000'}, // 配置代理
    contentBase: path.join(__dirname, 'public'), // 静态文件地址 boolean | string | array
    overlay: false, // 是否将编译器的错误显示在浏览器 boolean | object
    hot: true, // 启用热更新 搭配插件webpack.HotModuleReplacementPlugin()
    inline: true, // 使用内联模式，搭配热更新
  },

  // 8. 插件
  plugins: [new HtmlWebpackPlugin()],

  // 9. webpack的主目录(entry和loader相对于此目录解析)
  context: __dirname
}
```

## 修改配置达到更好的开发体验

1. 修改`resolve.extendssions`来修改自己想要的扩展名
2. 修改`resolve.alias`设置模块别名
3. 修改`devServer.proxy`解决跨域问题
4. 通过插件`webpack.definePlugin`注入`process.env.NODE_ENV`环境变量，然后在项目中判断这个的值来写出`devlopment`和`production`的兼容代码(实际就会编译成`if (false)`的形式，然后就会被`uglifyJS`删掉)

## 打包优化点

### 1.  公共类库抽离

```javascript
// 提取entry打包后的通用模块
entry: {
  entry1: './src/entry1.js',
  entry2: './src/entry2.js'
}
new webpack.optimize.CommonsChunkPlugin({
  name: 'common', // 指定公共类库的名字
  minChunks: function (moudle, count) {
      return (module.context && module.indexOf('node_modules') !== -1) || count >= 2
  }
})

// 明确第三方库
entry: {
  main: './src/index.js',
  vendor: ['lodash']
}
new webpack.optimize.CommonsChunkPlugin({
  name: 'vendor', 
  minChunks: function (moudle, count) {
      return (module.context && module.indexOf('node_modules') !== -1) || count >= 2
  }
})
```

### 2. 代码分割(懒加载)

```javascript
// import()中使用魔法注释指定chunk name
// vue中也会有自己的组件懒加载
import(/* webpackChunkName: "my-chunk-name" */ './main')

// 提取异步模块中的通用模块
entry: {
  main: './src/index.js',
  vendor: ['lodash']
}
new webpack.optimize.CommonsChunkPlugin({
  name: ['vendor'],
  minChunks: function (moudle, count) {
    return (module.context && module.indexOf('node_modules') !== -1) || count >= 2
  }
}),
new webpack.optimize.CommonsChunkPlugin({
  name: ['a', 'b'],
  async: 'async-common',
  minChunks: 2
})
```

### 3. 缓存

- 修改`output`的`filename`为`[name].[chunkhash].js`，`chunkFilename`为`[chunkhash].js`
- `chunkhash`表示的是根据文件内容生成的`hash`，所以只要文件内容改变，文件名就会改变，绕过缓存
- `webpack`会将每个模块的配置信息打包进`bundle`，由于配置信息可能会变导致文件名也变化，所以要提取`manifest`
- 假如新增或删除了模块，导致`module.id`顺序就变化，那一些没有修改的文件也会重新改名，可以使用`webpack.HashedModuleIdsPlugin`插件取代数字标识来解决这个问题

```javascript
new webpack.HashedModuleIdsPlugin(),
new webpack.optimize.CommonsChunkPlugin({
  name: ['vendor', 'manifest'],
  minChunks: function (moudle, count) {
    return (module.context && module.indexOf('node_modules') !== -1) || count >= 2
  }
}),
```

### 4. tree-shaking

- 通过`es6`的`import`静态分析，加上`uglifyJS`删除一些无用的代码
- 缺点
  - 对纯函数支持好但可能由于一些副作用导致不能删除一些代码
  - `babel`转义后的代码也可能变得不可删除

#### `webpack`和`rollup`

- 实际中发现`rollup`支持直接`import`时候就显示`tree-shaking`，`webpack`完全依赖`uglifyJS`
- `rollup`支持导出`es`模块，`webpack`不支持，只是导出`umd`的话，将无法进行`tree-shaking`
  - 可以通过将组件或模块打包成单独的文件或目录单独引入，但是比较麻烦，厂商一般就会提供一个`babel`插件可供按需加载
- `webpack`支持代码分割，`rollup`不支持

总结: `rollup`更适合打包工具库，组件库，`webpack`适合做工程项目

### 5. externals排除模块

不打包某些模块，手动通过`script`引入，用的比较少

## 杂项

1. 使用`html-webpack-plugins`生成`html`
2. 使用`extractTextPlugin`插件分离`css`
3. 使用`clean-webpack-plugin`插件清除文件夹

## 附带自配的react模板

```javascript
var path = require('path')
var webpack = require('webpack')
var HtmlWebpackPlugin = require("html-webpack-plugin")
var ExtractTextPlugin  = require("extract-text-webpack-plugin")
var CleanWebpackPlugin = require('clean-webpack-plugin')

module.exports = {
    entry: {
        main : './app/main.jsx',
        vendor : 'react'
    },
    output: {
        path: path.resolve(__dirname, './build'),
        publicPath: '/',
        filename: 'js/[name].[chunkhash].js',
        chunkFilename : "[chunkhash].js"
    },
    module: {
        rules: [
            {
                test: /\.jsx$/,
                loader: 'babel-loader'
            },
            {
                test: /\.js$/,
                loader: 'babel-loader',
                exclude: /node_modules/
            },
            {
                test: /\.(scss|css)$/,
                use : ExtractTextPlugin.extract({
                    use : [
                        {
                            loader : 'css-loader',
                            options : {
                                modules : true
                            }
                        },
                        {
                            loader : 'sass-loader'
                        }
                    ]
                })
            },
            {
                test: /\.(eot|svg|ttf|woff|woff2)(\?\S*)?$/,
                loader: 'file-loader?outputPath=fonts/'
            },
            {
                test: /\.(png|jpe?g|gif|svg)(\?\S*)?$/,
                loader: 'file-loader?outputPath=img/'
            }
        ]
    },
    // 插件
    plugins : [
        // 生成html插件
        new HtmlWebpackPlugin({
            title : '温叶react小项目',
            template : './template.ejs'
        }),
        // 分离css插件
        new ExtractTextPlugin('css/style.css'),
        // 替换module.id(为了缓存)
        new webpack.HashedModuleIdsPlugin(),
        // 提取公共库插件
        new webpack.optimize.CommonsChunkPlugin({
            name : ["vendor", "manifest"],            // 指定公共bundle的名称
            minChunks : function(moudle) {
                return module.context && module.indexOf("node_modules") !== -1;     // 设置引入的vendor存在于node_moudles目录中
            }
        }),
        // webpack3的范围提升插件
        new webpack.optimize.ModuleConcatenationPlugin()
    ],
    // 调试工具
    devtool: '//eval-source-map',
    // 解析模块
    resolve : {
        extensions : [".js", ".jsx", ".scss"],
        alias: {
            '@': path.resolve(__dirname, './app'),
            '@assets': path.resolve(__dirname, './app/assets'),
            '@components': path.resolve(__dirname, './app/components'),
            '@containers': path.resolve(__dirname, './app/containers'),
            '@store': path.resolve(__dirname, './app/store'),
            '@config': path.resolve(__dirname, './app/config'),
        }
    }
}

// 开发模式
if (process.env.NODE_ENV === 'development') {
    // 更改文件名
    module.exports.output.filename = 'js/[name].[hash].js';
    // 开发设置
    module.exports.devServer = {
        historyApiFallback: true,
        noInfo: true,
        contentBase : path.resolve(__dirname, 'build'),
        publicPath : '/',
        proxy : {
            "/json" : "http://localhost:4002",
            "/banner_pic" : "http://localhost:4002",
            "/list_pic" : "http://localhost:4002",
        },
        //hot : true,
    },
    // 新增插件
    module.exports.plugins = (module.exports.plugins || []).concat([
        // 热更新模块插件
        // new webpack.HotModuleReplacementPlugin(),
        // 定义环境变量插件
        new webpack.DefinePlugin({
            'process.env': {
            NODE_ENV: '"development"'
            }
        }),
    ])
}

// 生产模式
if (process.env.NODE_ENV === 'production') {
    // 改变调试模式
    module.exports.devtool = 'source-map'
    // 添加插件
    module.exports.plugins = (module.exports.plugins || []).concat([
        // 清除插件
        new CleanWebpackPlugin(['build']),
        // 定义环境变量插件
        new webpack.DefinePlugin({
            'process.env': {
            NODE_ENV: '"production"'
            }
        }),
        // 压缩插件
        new webpack.optimize.UglifyJsPlugin({
            compress: {
            warnings: false
            }
        }),
        // 升级插件
        new webpack.LoaderOptionsPlugin({
            minimize: true
        })
    ])
}
```
