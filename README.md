# vue的响应式原理总结

## 依赖技术

> 1. 利用Object.defineProperty劫持监听所有属性,每劫持一个对象分配一个发布者Dep
> `Dep.target`保证每次只有一个订阅目标

```js
    class Observer {
        constructor(data) {
        this.data = data

        Object.keys(data).forEach((key) => {
            this.defineReactive(this.data, key, data[key])
        })
        }
        defineReactive(data, key, val) {
        // 一个属性对应一个发布者
        const dep = new Dep()
        Object.defineProperty(data, key, {
            configurable: true,
            enumerable: true,
            get() {
            // 在订阅者第一次获取属性的时候,响应式添加到Dep
            if (Dep.target) {
                dep.addSub(Dep.target)
            }
            return val
            },
            set(newValue) {
            if (newValue === val) {
                return
            }
            val = newValue
            // 通知所有订阅者值已经改变
            dep.notify()
            },
        })
        }
    }
```

> 2.利用订阅发布者模式,实现所监听对象改变时通知订阅者相应改变
> 注意在Watcher中添加Dep.target,Watcher更新后需要置空,防止以后每次get属性时都生成一个订阅者

```js
    /* 订阅发布模式 */
    /* 发布者 */
    class Dep {
        constructor() {
        this.subs = []
        }
        addSub(sub) {
        this.subs.push(sub)
        }
        notify() {
        this.subs.forEach((sub) => {
            sub.update()
        })
        }
    }
    /* 订阅者 */
    class Watcher {
        constructor(node, name, vm) {
        this.node = node
        this.name = name
        this.vm = vm
        Dep.target = this
        // 取值调用get,在Observer响应式里面订阅属性对应的Dep
        this.update()
        // 赋空,只在get中添加一次订阅,不然每次普通的get会添加watcher
        Dep.target = null
        }
        update() {
        // console.log('Watcher-update')
        console.log(this.node)
        // 文本节点的监听.赋值到nodeValue
        if (this.node.nodeValue) {
            this.node.nodeValue = this.vm[this.name]
        }
        // input标签节点的监听,赋值value
        if (this.node.value) {
            this.node.value = this.vm[this.name]
        }
        }
    }
```

> 3.处理绑定的el元素
>  
> - 通过node.nodeType判断文本节点或者元素节点,两种节点监听属性后的赋值对象不一样
> - 创建DOM片段,遍历绑定的DOM结构(这里只模拟了一层树,多层需要递归遍历)
> - 双向绑定的实现:监听页面属性变化;监听input的输入事件

```js
        /* 渲染DOM */
      const reg = /\{\{(.*)\}\}/
      class Compiler {
        constructor(el, vm) {
          this.el = document.querySelector(el)
          this.vm = vm

          // // 创建DOM片段，让片段对象和
          this.frag = this._createFragment()
          this.el.appendChild(this.frag)
        }
        _createFragment() {
          const frag = document.createDocumentFragment()
          let child
          // 如果不使用appendChild,则每次都是第一个子节点
          // 使用frag用appendChild剪切每次的子节点,然后可以循环遍历
          while ((child = this.el.firstChild)) {
            this._compile(child) // 对每个node节点进行解析
            frag.appendChild(child)
          }
          return frag
        }
        _compile(node) {
          //标签元素节点,实现双向绑定
          if (node.nodeType === 1) {
            console.log(node.value)
            // console.log(node.attributes.value.nodeValue)
            const attrs = node.attributes
            if (attrs.hasOwnProperty('v-model')) {
              // 获取属性名称
              const name = attrs['v-model'].nodeValue
              // 监听输入事件,set方法调用发布者notify()
              node.addEventListener('input', (e) => {
                this.vm[name] = e.target.value
              })
              new Watcher(node, name, this.vm)
            }
          }
          //文本节点
          if (node.nodeType === 3) {
            if (reg.test(node.nodeValue)) {
              const name = RegExp.$1.trim()
              // 对使用该属性名称的文本对象进行监听
              new Watcher(node, name, this.vm)
            }
          }
        }
      }
```

> 4.vue类
> 代理_proxy作用:可以通过this.property操作this.$data.property

```js
      class Vue {
        constructor(options) {
          // 1.保存数据
          this.$options = options
          this.$data = options.data
          this.$el = options.el

          // 2.代理this.$data的数据
          // 所有this.$data.property可以通过this.property操作
          Object.keys(this.$data).forEach((key) => {
            this._proxy(key)
          })

          // 3.将data添加到响应式系统
          new Observer(this.$data)

          // 4.处理el
          new Compiler(this.$el, this)
        }

        _proxy(key) {
          Object.defineProperty(this, key, {
            configurable: true,
            enumerable: true,
            get() {
              return this.$data[key]
            },
            set(newValue) {
              this.$data[key] = newValue
            },
          })
        }
      }
```

> 5.测试的实例对象

```html
    <div id="app">
      <input id="test" type="text" v-model="message" value="test" />
      {{message}}
    </div>
        <script>
      const app = new Vue({
        el: '#app',
        data: {
          message: 'hello~',
        },
      })
    </script>
```
