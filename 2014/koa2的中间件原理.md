## context

各种挂载，例如挂载自己实现的请求响应方法和原生请求响应对象

## 中间件实现原理

- `app.use()`收集中间件，推进数组
- 将中间件数组用`compose`转换成洋葱函数(返回promise)
  - 获取第一个中间件并执行，第一参数为上下文，第二个参数为`next`函数，函数里面`dispatch`实现递归调用下一个中间件
- 请求时执行洋葱函数并响应

```javascript
// app.use()收集中间件，请求时候执行

const http = require('http')

// 相当于ctx.request
let request = {
  // ....
}
// 相当于ctx.response
let response = {
  // ...
}
// 相当于ctx(当不是最终版)
let context = {
  // 自定自己的一些方法
  // 代理request，response到自己身上
}

// 大名鼎鼎的compose函数 m1 -> next() -> m2 -> next() -> m3
function compose (middleware) {
  if (!Array.isArray(middleware)) throw new Error('middleware stack must be a array')
  for (const fn of middleware) {
    if (typeof fn !== 'function') throw new Error('Middleware must be composed of functions!')
  }

  // 返回包装成洋葱结构函数
  return function (context, next) {
    let index = -1 // 用来判断是否中间件多次使用await next()
    return dispatch(0)
    function dispatch (i) { // i就是数组下标
      if (i <= index) return Promise.reject(new Error('next() called multiple times'))
      index = i
      let fn = middleware[i] || next
      if (!fn) return Promise.resolve()
      try {
        // 指定第一个中间件，第二个参数是next()函数，递归调用下一个中间件
        // 返回包装后的promise，为了兼容普通函数
        return Promise.resolve(fn(context, function next () {
          return dispatch(i + 1)
        }))
      } catch (e) {
        return Promise.reject(e)
      }
    }
  }
}

// 响应函数
function respond (ctx) {
  // 其实有各种类型的操作(buffer, stream, string, json)，这里只考虑json
  // ...

  let body = ctx.body
  let res = ctx.res
  return res.end(JSON.stringify(body))
}

class Koa {
  constructor () {
    this.middleware = []
    this.context = Object.create(context)
    this.request = Object.create(request)
    this.response = Object.create(response)
  }

  listen (...args) {
    let server = http.createServer(this.callback()) // 每次接受请求都会执行callback()函数
    return server.listen(...args)
  }

  // use方法收集中间件，推进middleware数组
  use (fn) {
    if (typeof fn !== 'function') throw new TypeError('middleware must be a function!')
    this.middleware.push(fn)
    return this
  }

  callback () {
    // 得到洋葱函数
    const fn = compose(this.middleware)
    // 返回创建server用到的函数，req，res都是原生对象
    return (req, res) => {
      // 创建新的终极版ctx
      const ctx = this.createContext(req, res)
      // 调用洋葱函数执行中间件
      fn.call(ctx).then(() => {
        respond(ctx)
      }).catch(e => console.log(e))
    }
  }

  createContext (req, res) {
    const context = Object.create(this.context)
    // 各种互相挂载关系
    context.app = this
    context.res = res
    context.req = req
    // ....
    return context
  }
}
```

### async中间件使用demo

```javascript
let app = new Koa()
async function m1 (ctx, next) {
  console.log(1)
  await next()
  console.log(1)
}
async function m2 (ctx, next) {
  console.log(2)
  await next()
  console.log(2)
}
async function m3 (ctx, next) {
  console.log(3)
  await next()
  console.log(3)
}

app.use(m1)
app.use(m2)
app.use(m3)

app.listen(3000) 
//有请求时候输出:  1 2 3 3 2 1
```

### 普通函数中间件使用demo

```javascript
let app = new Koa()
function m1 (ctx, next) {
  console.log(1)
  next().then(() => {
    console.log(1)
  })
}
function m2 (ctx, next) {
  console.log(2)
  next().then(() => {
    console.log(2)
  })
}
function m3 (ctx, next) {
  console.log(3)
  next().then(() => {
    console.log(3)
  })
}

app.use(m1)
app.use(m2)
app.use(m3)

app.listen(3000)
//有请求时候输出:  1 2 3 3 2 1
```
