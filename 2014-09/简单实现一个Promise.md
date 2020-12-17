## 极简版

- 其实就是收集`then`方法回调，存进内部一个数组中，等到`resolve`调用，循环执行数组

```javascript
function Promise (fn) {
  // 不用this声明变量是不想暴露变量
  var value = null
  var callbacks = []

  // then方法
  this.then = function (onResolved) {
    callbacks.push(onResolved) // 将回调存储在callbacks数组中
    return this // 返回this实现链式调用
  }

  // resolve函数
  function resolve (val) {
    // setTimeout实现宏队列，确保then中的回调都已经收集完，再去遍历执行
    // 现实中的实现是微队列，这也就是无法完全模拟最真实promise的原因
    setTimeout(function () {
      callbacks.forEach(function (callback) {
        callback(val)
      })
    })
  }
  
  // 立即执行promise传入的函数
  fn(resolve)
}

let promise = new Promise((resolve) => {
  resolve('wenye')
})
promise.then(val => {
  console.log(val)
}).then(val => {
  console.log(val)
})

```

## 增强版

### 加入状态

- `then`回调中判断状态，如果是`pending`则将任务对象(见下文)推进数组，如果是其他则立即执行回调
- 在内部`resolve`或`reject`方法中个更改状态为`resolved`或`rejected`，并将参数值保存到`value`中

### 链式Promise

链式`promise`通过`then`方法中返回一个新的`promise`来实现，也就是下一个`then`是属于上一个返回(新的)`promise`

- 推进数组的是一个任务对象，包含`then`的回调以及返回`promise`的`resolve`函数
- 执行`then`的回调获取返回值，再传递给`resolve`函数执行，如果回调不存在，则直接使用之前的`value`传递给`resolve`函数执行。一旦`resolve`函数执行，下一个`then`的任务对象就会执行，依次实现链式

### 失败处理

`then`方法增加`onRejected`回调

### 异常处理

`try-catch`捕获`then`的回调，如果出错就调用`then`的`onRejected`回调

```javascript
// 每一个then是属于上一次promise的，一个promise搭配一个then

function Promise (fn) {
  var callbacks = []
  var value = null
  var state = 'pending'

  // 链式收集回调
  this.then = function (onResolved, onRejected) {
    return new Promise(function (resolve, reject) {
      // 处理任务对象
      handle({
        onResolved: onResolved || null, // 本身的回调
        onRejected: onRejected || null,
        resolve: resolve, // 本身返回promise的resolve函数，可以用来调用后一个任务对象
        reject: reject
      })
    })
  }

  function handle (callback) {
    // A部分: new Promise后执行
    if (state === 'pending') {
      callbacks.push(callback)
      return
    }

    // B部分: 异步resolve后执行
    var cb = state === 'resolved' ? callback.onResolved : callback.onRejected
    if (cb === null) {
      cb = state === 'resolved' ? callback.resolve : callback.reject
      cb(value)
      return
    }
    try {
      var ret = cb(value) // 获取前一个then的执行结果
      callback.resolve(ret) // 调用后一个then的resolve
    } catch (e) {
      callback.reject(e)
    }
  }

  function resolve (val) {
    // resolve的值为promise实例
    if (val && (typeof val === 'object' || typeof val === 'function')) { // 为了兼容其他promise实现
      var then = val.then
      if (typeof then === 'function') {
        then.call(val, resolve, reject) // 这个resolve就是onResolved
        return
      }
    }

    // resolve的值为非promise实例
    value = val
    state = 'resolved'
    execute()
  }

  function reject (reason) {
    state = 'rejected'
    value = reason
    execute()
  }

  function execute () {
    setTimeout(function () {
      callbacks.forEach(function (callback) {
        handle(callback)
      }, 0)
    })
  }

  // new Promise的回调会立刻执行
  fn(resolve, reject)
}

let test = new Promise((resolve, reject) => {
  setTimeout(function () {
    resolve('wenye')
  }, 1000)
})

test.then((val) => {
  return new Promise((resolve, reject) => {
    setTimeout(function () {
      resolve('wenye')
    }, 1000)
  })
}).then(val => {
  console.log(val)
}, err => {
  console.log(err.message)
})

```

## 按照PromiseA+规范完善

- 比如支持`thenable`，从而兼容其他版本的`promise`

实际上无法真正实现浏览器一样的`promise`，因为它使用的是微任务，而我们只能使用宏任务