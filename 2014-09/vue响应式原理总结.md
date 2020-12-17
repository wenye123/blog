## 前言

`vue`源码主要有四块内容，响应式系统，执行流程，模板编译，虚拟DOM

## 响应式原理

### 依赖收集

- 通过`Object.defineProperty`方法将`data`中的值转换成`getter`和`setter`的形式。`getter`中收集依赖，`setter`中触发依赖
- 依赖就是`watcher`，在`vue`中有三种情况，`computed`，`watch`，`模板中的引用`
- 如何收集依赖?
  - 定义一个`Dep`类，里面有一个`subs`数组，`depend`方法(如果`Dep.target`存在，就将`Dep.target`添加进subs数组中)
  - 在`Object.defineProperty`实例化一个`dep`，在`getter`中调用`depend`方法，也就是说一旦调用`getter`方法被调用，该键就会收集依赖
  - 定义一个`Watch`类，初始化中就设置`Dep.target`为`this`，然后再调用那个键的`getter`，这样子就触发`getter`里的`depend`方法，完成依赖收集
- 也就是说很多地方通过`new Watch`实例化的依赖都会被添加到那个键的`dep`中，并在`setter`被调用(值发生改变)时候循环触发

### 将data中的键转换成getter/setter也有两种形式

- 对象: 遍历将子元素转换成`getter`/`setter`
- 数组: 修改数组方法(也有两种情况)，加个触发依赖的钩子
  - 这里有个难点就是在数组中如何通知`dep`，其实是通过给`key-value`中`value`加上`__ob__`属性指向本身的`Observer`实例，`Observer`实例本身也有个`dep`，也就是键值对的键值都存储了一个`dep`，`value`可以通过`value.__ob__.dep`来实现触发依赖(很麻烦，得看源码才能懂)

### 代理data中的值到vm

其实就是给`vm`定义`getter`和`setter`，指向`data`中的值

### 总之很是复杂，还是来看看我抽象出来的代码吧

```javascript
// 工具函数: 移除数组中指定元素
function remove (arr, item) {
  if (arr.length) {
    const index = arr.indexOf(item)
    if (index > -1) {
      return arr.splice(index, 1)
    }
  }
}
// 工具函数: 解析路径，比如parsePath('wenye.name')({wenye: {name: 'xxx'}}) => xxx
function parsePath (path) {
  const segments = path.split('.')
  return function (obj) {
    for (let i = 0; i < segments.length; i++) {
      if (!obj) return
      obj = obj[segments[i]]
    }
    return obj
  }
}
// 工具函数: 定义对象的一个键值
function def (obj, key, val, enumerable) {
  Object.defineProperty(obj, key, {
    value: val,
    enumerable: !!enumerable,
    writable: true,
    configurable: true
  })
}

// 依赖类
class Dep {
  constructor () {
    // 存储观察者
    this.subs = []
  }

  addSub (sub) {
    this.subs.push(sub)
  }

  removeSub (sub) {
    remove(this.subs, sub)
  }

  // 依赖收集
  depend () {
    if (Dep.target) {
      this.addSub(Dep.target)
    }
  }

  // 遍历数组触发依赖
  notify () {
    const subs = this.subs.slice()
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update() // 调用观察者的update方法
    }
  }
}

// 观察者类
// 一实例化就将此观察者实例添加进expOrFn的依赖
// 一般三处地方需要实例化: 模板，computed，watch
class Watch {
  constructor (vm, expOrFn, cb) {
    this.vm = vm
    this.getter = parsePath(expOrFn)
    this.cb = cb
    this.value = this.get()
  }

  // 收集依赖
  get () {
    // 将Dep.target设置为this，为了收集
    Dep.target = this
    // 调用getter实现收集依赖
    let value = this.getter.call(this.vm, this.vm)
    // 依赖手机完，将Dep.target设置为undefined
    Dep.target = undefined
    return value
  }

  // 调用回调
  update () {
    const oldValue = this.value
    this.value = this.get() // 这里再次调用this.get()会重新收集依赖，实际中依赖应该做去重处理，这里就不做了
    this.cb.call(this.vm, this.value, oldValue)
  }
}

// Observer类(观察一整个data)
class Observer {
  constructor (value) {
    this.value = value
    this.dep = new Dep() // 这个dep是给数组用的，通过value.__ob__.dep就能访问到，而对象则是key的getter就能访问到dep
    def(value, '__ob__', this) // 给value(key-value)定义__ob__证明已经observer过了
    if (Array.isArray(value)) { // 数组的observer处理
      // 改造数组方法
      if ('__proto__' in {}) { // 如果存在__proto__属性则直接指向arrayMethods
        value.__pro__ = arrayMethods
      } else { // 否则定义定义方法
        for (let i = 0; i < arrayKeys.length; i++) {
          const key = arrayKeys[i]
          def(value, key, arrayMethods[key])
        }
      }
      // observer数组的每一项
      this.observerArray(value)
    } else { // 对象的observer处理
      this.walk(value)
    }
  }

  // 遍历对象中的键进行响应式处理
  walk (obj) {
    const keys = Object.keys(obj)
    for (let i = 0, l = keys.length; i < l; i++) {
      defineReactive(obj, keys[i], obj[keys[i]])
    }
  }

  // 给数组的每一个成员都重新new Observer
  observerArray (arr) {
    for (let i = 0; i < arr.length; i++) {
      /* eslint-disable no-new */
      new Observer(arr[i])
    }
  }
}

// 函数: 定义响应式
function defineReactive (data, key, value) {
  let dep = new Dep()
  let valueOb = observer(value) // 获取value的__ob__
  Object.defineProperty(data, key, {
    enumerable: true,
    configurable: true,
    get () { // getter中收集依赖(key的依赖以及value的依赖)
      // key收集依赖
      dep.depend()
      // value收集依赖
      if (valueOb) {
        valueOb.dep.depend()
      }
      // 如果value是数组，子元素也要收集依赖
      if (Array.isArray(value)) {
        dependArray(value)
      }
      return value
    },
    set (newVal) { // setter中触发依赖
      if (value === newVal) {
        return
      }
      valueOb = observer(newVal) // 新的值重新observer，确保响应式
      dep.notify()
      value = newVal
    }
  })
}

// 函数: 返回value(key-value)上的__ob__，没有则创建
function observer (value) {
  // 不是对象则返回
  if (value !== null && typeof value === 'object') {
    return
  }
  let ob = null
  if (Object.prototype.hasOwnProperty.call(value, '__ob__')  && value.__ob__ instanceof Observer) {
    ob = value.__ob__
  } else if (Array.isArray(value) || Object.prototype.toString.call(value) === '[object Object]') { // 是数组或原生对象
    ob = new Observer(value)
  }
  return ob
}

// 函数: 数组的每个元素(value)都要收集依赖
function dependArray (value) {
  for (let item, i = 0, l = value.length; i < l; i++) {
    item = value[i]
    item && item.__ob__ && item.__ob__.dep.depend()
    // 如果子元素是数组则递归
    if (Array.isArray(item)) {
      dependArray(item)
    }
  }
}

// 自定义一套数组方法arrayMethods,添加响应式功能用于重写data中的数组
const arrayProto = Array.prototype
const arrayMethods = Object.create(arrayProto)
const arrayKeys = Object.getOwnPropertyNames(arrayMethods)
;['push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse'].forEach(function (method) {
  const original = arrayProto[method] // 存储原始数组方法
  def(arrayMethods, method, function mutator (...args) {
    const result = original.apply(this, args) // 执行原生方法获取结果数组
    const ob = this.__ob__ // 获取value的__ob__
    // 获取新增的值(数组)并observer
    let inserted = []
    switch (method) {
      case 'push':
      case 'unshift':
        inserted = args
        break
      case 'splice':
        inserted = args.slice(2)
        break
    }
    if (inserted) ob.observerArray(inserted)
    // 通知数组因为调用方法而发生改变
    ob.dep.notify()
    return result
  })
})

/** demo演示 */

// 代理函数
function _proxy (data, context) {
  Object.keys(data).forEach(key => {
    Object.defineProperty(context, key, {
      configurable: true,
      enumerable: true,
      get: function proxyGetter () {
        return context.data[key]
      },
      set: function proxySetter (val) {
        context.data[key] = val
      }
    })
  })
}

class Vue {
  constructor (options) {
    this.data = options.data
    this.watch = options.watch

    // 代理
    _proxy(this.data, this)
    // 观察data
    new Observer(this.data)
    // 遍历watch作为watcher
    Object.keys(this.watch).forEach(key => {
      new Watch(this, key, this.watch[key])
    })
  }
}

let app = new Vue({
  el: '#app',
  data: {
    name: 'wenye',
    age: 22
  },
  watch: {
    name (val) {
      console.log(val)
    }
  }
})

```

### 注意

上面的代码也是省略了很多细节的，比如

- `watch`的去重
- 不仅仅是`data`的`dep`保存了`watch`，`watch`中也保存了`dep`

例如在计算属性中，`computed`有一个`watch`被添加进了依赖`data`的`dep`中，一旦其中一个依赖的`data`发生改变就会执行到`computed`的`watch`，然后`computed`的`watch`就会把`watch.dirty`属性设置为`true`，但不是执行回调求值，而是在对`comouted`进行`getter`时判断`watch.dirty`属性是否为`true`才触发求值操作(懒加载)。

还有点就是为什么计算属性依赖的`data`发生改变，计算属性的模板DOM也会发生改变？

因为计算属性的`watch.deps`保存了依赖`data`的`dep`，于是将计算属性模板`watch`也添加进`data`的`dep`中去，所以依赖`data`的值改变，也会触发计算属性的模板`watch`重新求值

- ....