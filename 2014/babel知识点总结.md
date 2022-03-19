## 1. 安装包总结

1. `babel-cli`是命令行转码，自带有`babel-node`可以直接运行`es6`文件代码
2. `babel-core`编程方式使用`babel`，搭配`webpack`时候需要安装
3. `babel-register`调用后重写`require`命令，`require`之前先转码，不推荐在生产环境中用

## 2. 搭配webpack

```javascript
// webpack
npm install --save-dev babel-core babel-preset-env babel-plugin-transform-runtime
npm install --save babel-runtime
rules: [
  {
    test: '/\.js$/',
    loader: 'babel-loader'
  }
]

//.babelrc(babel的配置文件)
{
  // 预设
  "presets": [
    ["env", {
      "modules": false,
      "target": {
        "node": "current"
      },
      "loose": true
    }]
  ],
  "plugins": [
    "transform-runtime",
    // 支持宽松模式，比如es6类的方法是不可枚举的(Object.defindProperty实现)
    // 采用此模式则可以(挂载到类的prototype)
    ["transform-es2015-classes", { "loose": true }]
  ] // 插件
}
```

### presets预设

- 一般使用`env`，可以设置环境类型，根据环境确定是否要转换，默认行为是`lastest`
  - 记得将`modules`设置为`false`搭配构建工具，不转码`modules`
  - `target`目标 设置`browse`r或`node`的范围
  - `loose`支持宽松模式
- 除此之外还有`react`和`flow`以及草案阶段`stage-x`(0-4， 值越小内容越多)，看情况使用，可以连用

### 编译顺序

- `plugins`优先于`presets`
- `plugins`按照从左向右编译
- `presets`按照从右向左编译，因为作者认为会写成['es2015', 'stage-0']

## 3. babel-polyfill和babel-runtime的区别

- `babel-polyfill`在文件头直接`import`，直接修改全局变量，缺点会污染全局变量，而且会全局打包进`bundle`
- `babel-runtime`是分散的`polyfill`模块，可以单独引入自己想要的`helper`，缺点是手动引入效率低，可以使用插件`babel-plugin-transform-runtime`自动化处理，但是还是有缺点是无法识别实例方法如`[1, 2, 3].includes()`
