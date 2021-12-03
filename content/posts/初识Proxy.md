---
title: "初识Proxy"
date: 2021-12-03T00:49:31+08:00
draft: false
---

## 初识Proxy

最近在复习Vue，不可避免的会接触到vue3，所以也不可避免的会思考这些问题

1. vue3实现响应式为什么要使用proxy替换Object.defineProperty？Proxy对比Object.defineProperty有啥优缺点？
2. 怎么通过Proxy实现响应式？

本文会回答这两个问题，通过这些问题探讨Proxy，以及Proxy在日常开发中的应用场景。

## 认识Proxy

Proxy意思翻译过来就是**代理**，外界对目标对象的访问都会被Proxy拦截，从而可以实现基本操作的拦截和自定义。

#### 用法

```javascript
let proxy = new Proxy(target,handler)
```

- target: 所要拦截的目标对象
- handler: handler是一个包含你要拦截和处理的对象，当对象被代理时，handler通过捕捉器(trap)实现对各种行为的拦截

目前proxy支持13种行为的拦截

|       handler方法        |                           何时触发                           |
| :----------------------: | :----------------------------------------------------------: |
|           get            |                           读取属性                           |
|           set            |                           写入属性                           |
|           has            |                           in操作符                           |
|      deleteProperty      |                         delete操作符                         |
|          apply           |                           函数调用                           |
|        construct         |                          new操作符                           |
|      getPrototypeOf      |                    Object.getPrototypeOf                     |
|      setPrototypeOf      |                    Object.setPrototypeOf                     |
|       isExtensible       |                     Object.isExtensible                      |
|    preventExtensions     |                   Object.preventExtensions                   |
|      defineProperty      |     Object.defineProperty,<br />Object.defineProperties      |
| getOwnPropertyDescriptor | Object.getOwnPropertyDescriptor,<br />for...in,<br />Object.keys/values/entries |
|         ownKeys          | Object.getOwnPropertyNames,<br />Object.getOwnPropertySymbols,<br />for...in,<br />Object.keys/values/entries |

### Reflect

reflect翻译过来是映射的意思，在MDN上是这样定义的

> **Reflect** 是一个内置的对象，它提供拦截 JavaScript 操作的方法。

每个可用的代理捕捉器(trap)都有一个对应的同名Reflect函数，并能产生相同的行为。

```javascript
let obj = {
    a: 10,
    name: 'oyc'
}

let newTarget = new Proxy(obj, {
    set(target, key, val) {
        console.log(`Set ${key}=${val} `);
    }
})

// newTarget.a = 20;  //Set a=20
// Reflect.set(newTarget, 'a', 20); //Set a=20

newTarget.name = 'oyq'; //Set name=oyq

Reflect.set(newTarget, 'name', 'oyq'); //Set name=oyq
```

从这可以看出，Reflect和trap表现出来的行为是相同的。所以当你为如何去触发trap而烦恼的时候，也许这个Reflect可以帮到你。



## 两个问题

大致学习完proxy的内容后，再来尝试解答下页头提到的两个个问题。

#### vue3实现响应式为什么要使用proxy替换Object.defineProperty？优缺点？

**优点**

- **性能更好**,Object.defineProperty只能劫持对象的属性，所以如果有嵌套对象，初始化时需要遍历data中的每个属性,在vue3中，proxy可以代理对象，不需要像vue2那样对属性进行遍历的操作

```javascript
//vue2
function reactive(obj) {
    // 遍历对象
    for (const item in obj) {
        if (obj[item] && typeof obj[item] == 'object') {
            // 递归，又重新遍历
            reactive(obj[item])
        } else {
            defineReactive(obj, item, obj[item])
        }
    }
}

function defineReactive(obj, key, val) {
    Object.defineProperty(obj, key, {
        //set,get操作
    })
}


//vue3
let newTarget = new Proxy(obj, {
  // set,get操作
})
```



- **自动代理新增属性,数组**,Object.defineProperty的实现是对属性进行劫持，所以当新增属性时，需要重新遍历，对新增的重新进行劫持。所以需要vue2对新增的属性，以及数组进行**$set**才能保证属性是响应式的，这个过程是手动的。

```javascript
let obj = {
  a: 10,
  name: 'oyc'
}
//vue2
this.$set(this.obj, 'age', 18); //每次新增都需要进行这个操作


//vue3
//自动代理
let newTarget = new Proxy(obj, {
  get(target, key) {
    return Reflect.get(target, key);
  },
  set(target, key, val) {
    return Reflect.set(target, key, val);
  }
})
```

- Proxy支持13中拦截操作，Object.defineProperty无法比拟
- Proxy是新标准，后续也会优先优化，Object.defineProperty的setter,getter后续应该优化的优先级较低

**缺点**

显而易见的，Proxy的兼容性相较于Object.defineProperty，是较低的，不支持IE浏览器。不过以目前的市场份额来看，IE浏览器的市场份额也不多，目前微软也将ie换成了chrome内核的edge，所以激进点的项目是完全可以使用proxy的。

![image-20211202224940291.png](https://i.loli.net/2021/12/02/uU7jydJ5YVntrLv.png)

#### 怎么通过Proxy实现响应式？

```javascript
let obj1 = {
    a: 10,
    name: 'John',
    list: [1, 2, 3],
    obj2: {
        obj3: 'oo',
        obj4: {
            name: 'oyc'
        }
    }
}

// 判断是否是对象
const isObj = (obj) => typeof obj === 'object' && obj !== null;

const render = (key, val) => {
    console.log(`Render ${key}=${val}`);
}

function reactive(obj) {
    if (!isObj(obj)) {
        return obj;
    }
    const handler = {
        get(target, key) {
            // 对嵌套对象遍历
            if (isObj(target[key])) {
                // 递归
                return reactive(target[key]);
            }
            return Reflect.get(target, key);
        },
        set(target, key, val) {
            // 渲染
            render(key, val);
            return Reflect.set(target, key, val);
        }
    }
    const targetProxyObj = new Proxy(obj, handler);
    return targetProxyObj
}

let myObj = reactive(obj1);

myObj.a = 20; // Render a=20
myObj.b = 30; //新增属性 Render b=30
myObj.list = [1, 2, 5, 6]; //修改数组 //Render list=1,2,5,6
myObj.obj2.obj4.name = 'oyq'; //修改嵌套对象 //Render name=oyq
```

## Proxy应用场景

- 写入默认值,日常开发经常碰到 **ReferenceError: xxx is not defined** 这种错误，这里我们可以代理，当属性不存在时，不报错，而设置一个默认值

```javascript
let obj = {
    name: 'oyq'
}

let proxyObj = new Proxy(obj, {
    get(target, key) {
        if (Reflect.has(target, key)) {
            return target[key];
        } else {
            return 'OYC';
        }
    },
})

console.log(proxyObj.age);//OYC
```

- 用Proxy来包装fetch,让fetch更易用

```javascript
let handlers = {
  get (target, property) {
    if (!target.init) {
      // 初始化对象
      ['GET', 'POST'].forEach(method => {
        target[method] = (url, params = {}) => {
          return fetch(url, {
            headers: {
              'content-type': 'application/json'
            },
            mode: 'cors',
            credentials: 'same-origin',
            method,
            ...params
          }).then(response => response.json())
        }
      })
    }

    return target[property]
  }
}
let API = new Proxy({}, handlers)

await API.GET('XXX')
await API.POST('XXX', {
  body: JSON.stringify({name: 1})
})
```

- 检验表单

```javascript
let formData = {
    name: '',
    age: ''
}

let proxyObj = new Proxy(formData, {
    set(target, key, val) {

        if (key === 'age' && typeof val !== 'number') {
            console.log('age must be number');
        }

        if (key === 'name' && typeof val !== 'string') {
            console.log('name must be string');
        }
    }
})

proxyObj.age = 'oyc'; //age must be number
proxyObj.name = 18; //name must be string
```

- 负索引数组

```javascript
let arr = [1, 2, 3, 4, 5]

let proxyArray = new Proxy(arr, {
    get(target, key) {
        const index = key < 0 ? target.length + Number(key) : key;
        return Reflect.get(target, index)
    }
})

console.log(proxyArray[-1]); //5
```

