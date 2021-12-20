---
title: "详解虚拟DOM与Diff算法"
date: 2021-12-20T23:11:34+08:00
draft: true
---

## 详解虚拟DOM与Diff算法

最近复习到虚拟DOM与Diff，翻阅了众多资料，特此总结了这篇长文，加深自己对vue的理解。这篇文章比较详细的分析了vue的虚拟DOM,Diff算法，其中一些关键的地方从别处搬运了一些图进行说明(感谢制图的大佬)，也包含比较详细的源码解读。

### 真实DOM的渲染

在讲虚拟DOM之前，先说一下真实DOM的渲染。

![](https://upload-images.jianshu.io/upload_images/15050783-c14e5c2c54996b48.png?imageMogr2/auto-orient/strip|imageView2/2/w/646/format/webp)



浏览器真实DOM渲染的过程大概分为以下几个部分

1. **构建DOM树**。通过HTML parser解析处理HTML标记，将它们构建为DOM树(DOM tree)，当解析器遇到非阻塞资源(图片，css),会继续解析，但是如果遇到script标签(特别是没有async 和 defer属性)，会阻塞渲染并停止html的解析，这就是为啥最好把script标签放在body下面的原因。
2. **构建CSSOM树**。与构建DOM类似,浏览器也会将样式规则，构建成CSSOM。浏览器会遍历CSS中的规则集，根据css选择器创建具有父子，兄弟等关系的节点树。
3. **构建Render树**。这一步将DOM和CSSOM关联，确定每个 DOM 元素应该应用什么 CSS 规则。将所有相关样式匹配到DOM树中的每个可见节点，并根据CSS级联确定每个节点的计算样式，不可见节点(head,属性包括 display:none的节点)不会生成到Render树中。
4. **布局/回流(Layout/Reflow)**。浏览器第一次确定节点的位置以及大小叫布局，如果后续**节点位置以及大小发生变化**，这一步触发布局调整，也就是 **回流**。
5. **绘制/重绘(Paint/Repaint)**。将元素的每个可视部分绘制到屏幕上，包括文本、颜色、边框、阴影和替换的元素（如按钮和图像）。如果**文本、颜色、边框、阴影**等这些元素发生变化时，会触发**重绘(Repaint)**。为了确保重绘的速度比初始绘制的速度更快，屏幕上的绘图通常被分解成数层。将内容提升到GPU层(可以通过tranform,filter,will-change,opacity触发)可以提高绘制以及重绘的性能。
6. **合成(Compositing)**。这一步将绘制过程中的分层合并，确保它们以正确的顺序绘制到屏幕上显示正确的内容。



### 为啥需要虚拟DOM

上面这是一次DOM渲染的过程，如果dom更新，那么dom需要重新渲染一次，如果存在下面这种情况

```javascript
<body>
    <div id="container">
        <div class="content" style="color: red;font-size:16px;">
            This is a container
        </div>
				....
        <div class="content" style="color: red;font-size:16px;">
            This is a container
        </div>
    </div>
</body>
<script>
    let content = document.getElementsByClassName('content');
    for (let i = 0; i < 1000000; i++) {
        content[i].innerHTML = `This is a content${i}`;
        // 触发回流
        content[i].style.fontSize = `20px`;
    }
</script>
```

那么需要真实的操作DOM100w次,触发了回流100w次。每次DOM的更新都会按照流程进行无差别的真实dom的更新。所以造成了很大的性能浪费。如果循环里面是复杂的操作，频繁触发回流与重绘，那么就很容易就影响性能，造成卡顿。另外这里要说明一下的是，虚拟DOM并不是意味着比DOM就更快，性能需要分场景，虚拟DOM的性能跟模板大小是正相关。虚拟DOM的比较过程是不会区分数据量大小的，在组件内部只有少量动态节点时，虚拟DOM依然是会对整个vdom进行遍历，相比直接渲染而言是多了一层操作的。

```javascript
	<div class="list">
    <p class="item">item</p>
    <p class="item">item</p>
    <p class="item">item</p>
    <p class="item">{{ item }}</p>
    <p class="item">item</p>
    <p class="item">item</p>
  </div>
```

比如上面这个例子，虚拟DOM。虽然只有一个动态节点，但是虚拟DOM依然需要遍历diff整个list的class，文本，标签等信息，最后依然需要进行DOM渲染。如果只是dom操作，就只要操作一个具体的DOM然后进行渲染。虚拟DOM最核心的价值在于，它能通过js描述真实DOM，表达力更强，通过声明式的语言操作，为开发者提供了更加方便快捷开发体验，而且在没有手动优化，大部分情景下，保证了性能下限，性价比更高。

### 虚拟DOM

虚拟DOM本质上是一个js对象，通过对象来表示真实的DOM结构。tag用来描述标签，props用来描述属性，children用来表示嵌套的层级关系。

```javascript
const vnode = {
    tag: 'div',
    props: {
        id: 'container',
    },
    children: [{
        tag: 'div',
        props: {
            class: 'content',
        },
      	text: 'This is a container'
    }]
}

//对应的真实DOM结构
<div id="container">
  <div class="content">
    This is a container
  </div>
</div>
```

虚拟DOM的更新不会立即操作DOM，而是会通过diff算法，找出需要更新的节点，按需更新，并将更新的内容保存为一个js对象，更新完成后再挂载到真实dom上，实现真实的dom更新。通过虚拟DOM，解决了操作真实DOM的三个问题。

1. 无差别频繁更新导致DOM频繁更新，造成性能问题
2. 频繁回流与重绘
2. 开发体验

另外由于虚拟DOM保存的是js对象，天然的具有**跨平台**的能力,而不仅仅局限于浏览器。

#### 优点

总结起来，虚拟DOM的优势有以下几点

1. 小修改无需频繁更新DOM，框架的diff算法会自动比较，分析出需要更新的节点，按需更新
2. 更新数据不会造成频繁的回流与重绘
2. 表达力更强，数据更新更加方便
3. 保存的是js对象，具备跨平台能力

#### 不足

虚拟DOM同样也有缺点，首次渲染大量DOM时，由于多了一层虚拟DOM的计算，会比innerHTML插入慢。

### 虚拟DOM实现原理

主要分三部分

1. 通过js建立节点描述对象
2. diff算法比较分析新旧两个虚拟DOM差异
3. 将差异patch到真实dom上实现更新

#### Diff算法

为了避免不必要的渲染，按需更新，虚拟DOM会采用Diff算法进行虚拟DOM节点比较,比较节点差异，从而确定需要更新的节点，再进行渲染。vue采用的是**深度优先，同层比较**的策略。

![](https://img2018.cnblogs.com/blog/1015847/201901/1015847-20190102105213116-466136499.png)

新节点与旧节点的比较主要是围绕三件事来达到渲染目的

1. **创建新节点**
2. **删除废节点**
3. **更新已有节点**

**如何比较新旧节点是否一致呢？**

```javascript
function sameVnode(a, b) {
    return (
        a.key === b.key &&
        a.asyncFactory === b.asyncFactory && (
            (
                a.tag === b.tag &&
                a.isComment === b.isComment &&
                isDef(a.data) === isDef(b.data) &&
                sameInputType(a, b) //对input节点的处理
            ) || (
                isTrue(a.isAsyncPlaceholder) &&
                isUndef(b.asyncFactory.error)
            )
        )
    )
}

//判断两个节点是否是同一种 input 输入类型
function sameInputType(a, b) {
    if (a.tag !== 'input') return true
    let i
    const typeA = isDef(i = a.data) && isDef(i = i.attrs) && i.type
    const typeB = isDef(i = b.data) && isDef(i = i.attrs) && i.type
    //input type 相同或者两个type都是text
    return typeA === typeB || isTextInputType(typeA) && isTextInputType(typeB)
}
```

可以看到，两个节点是否相同是需要比较**标签(tag)**，**属性(在vue中是用data表示vnode中的属性props)**, **注释节点(isComment)**的,另外碰到input的话，是会做特殊处理的。

#### 创建新节点

当新节点有的，旧节点没有，这就意味着这是全新的内容节点。只有元素节点，文本节点，注释节点才能被创建插入到DOM中。

#### 删除旧节点

当旧节点有，而新节点没有，那就意味着，新节点放弃了旧节点的一部分。删除节点会连带的删除旧节点的子节点。

#### 更新节点

新的节点与旧的的节点都有，那么一切以新的为准，更新旧节点。如何判断是否需要更新节点呢?

- 判断新节点与旧节点是否完全一致,一样的话就不需要更新

```javascript
  // 判断vnode与oldVnode是否完全一样
  if (oldVnode === vnode) {
    return;
  }
```

- 判断新节点与旧节点是否是静态节点，key是否一样，是否是克隆节点(如果不是克隆节点，那么意味着渲染函数被重置了，这个时候需要重新渲染)或者是否设置了once属性,满足条件的话替换componentInstance

```javascript
  // 是否是静态节点，key是否一样，是否是克隆节点或者是否设置了once属性
  if (
    isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
  ) {
    vnode.componentInstance = oldVnode.componentInstance;
    return;
  }
```

- 判断新节点是否有文本(通过text属性判断)，如果有文本那么需要比较同层级旧节点，如果旧节点文本不同于新节点文本，那么采用新的文本内容。如果新节点没有文本，那么后面需要对子节点的相关情况进行判断

```javascript
//判断新节点是否有文本
if (isUndef(vnode.text)) {
  //如果没有文本，处理子节点的相关代码
  ....
} else if (oldVnode.text !== vnode.text) {
  //新节点文本替换旧节点文本
  nodeOps.setTextContent(elm, vnode.text)
}
```

- 判断新节点与旧节点的子节点相关状况。这里又能分为4种情况
  1. 新节点与旧节点**都有子节点**
  2. **只有新节点有**子节点
  3. **只有旧节点有**子节点
  4. 新节点与旧节点**都没有子节点**

**都有子节点**

对于都有子节点的情况，需要对新旧节点做比较，如果他们不相同，那么需要进行diff操作，在vue中这里就是updateChildren方法，后面会详细再讲，子节点的比较主要是双端比较。

```javascript
//判断新节点是否有文本
if (isUndef(vnode.text)) {
    //新旧节点都有子节点情况下，如果新旧子节点不相同，那么进行子节点的比较，就是updateChildren方法
    if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
    }
} else if (oldVnode.text !== vnode.text) {
    //新节点文本替换旧节点文本
    nodeOps.setTextContent(elm, vnode.text)
}
```

**只有新节点有子节点**

只有新节点有子节点，那么就代表着这是新增的内容，那么就是新增一个子节点到DOM，新增之前还会做一个重复key的检测，并做出提醒，同时还要考虑，旧节点如果只是一个文本节点，没有子节点的情况，这种情况下就需要清空旧节点的文本内容。

```javascript
//只有新节点有子节点
if (isDef(ch)) {
  //检查重复key
  if (process.env.NODE_ENV !== 'production') {
    checkDuplicateKeys(ch)
  }
  //清除旧节点文本
  if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
  //添加新节点
  addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
}

//检查重复key
function checkDuplicateKeys(children) {
  const seenKeys = {}
  for (let i = 0; i < children.length; i++) {
      const vnode = children[i]
      //子节点每一个Key
      const key = vnode.key
      if (isDef(key)) {
          if (seenKeys[key]) {
              warn(
                  `Duplicate keys detected: '${key}'. This may cause an update error.`,
                  vnode.context
              )
          } else {
              seenKeys[key] = true
          }
      }
  }
}
```

**只有旧节点有子节点**

只有旧节点有，那就说明，新节点抛弃了旧节点的子节点，所以需要删除旧节点的子节点

```javascript
if (isDef(oldCh)) {
  //删除旧节点
  removeVnodes(oldCh, 0, oldCh.length - 1)
}
```

**都没有子节点**

这个时候需要对旧节点文本进行判断，看旧节点是否有文本，如果有就清空

```javascript
if (isDef(oldVnode.text)) {
  //清空
  nodeOps.setTextContent(elm, '')
}
```

整体的逻辑代码如下

```javascript
function patchVnode(
    oldVnode,
    vnode,
    insertedVnodeQueue,
    ownerArray,
    index,
    removeOnly
) {
    // 判断vnode与oldVnode是否完全一样
    if (oldVnode === vnode) {
        return
    }

    if (isDef(vnode.elm) && isDef(ownerArray)) {
        // 克隆重用节点
        vnode = ownerArray[index] = cloneVNode(vnode)
    }

    const elm = vnode.elm = oldVnode.elm

    if (isTrue(oldVnode.isAsyncPlaceholder)) {
        if (isDef(vnode.asyncFactory.resolved)) {
            hydrate(oldVnode.elm, vnode, insertedVnodeQueue)
        } else {
            vnode.isAsyncPlaceholder = true
        }
        return
    }
		// 是否是静态节点，key是否一样，是否是克隆节点或者是否设置了once属性
    if (isTrue(vnode.isStatic) &&
        isTrue(oldVnode.isStatic) &&
        vnode.key === oldVnode.key &&
        (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
        vnode.componentInstance = oldVnode.componentInstance
        return
    }

    let i
    const data = vnode.data
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
        i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children

    if (isDef(data) && isPatchable(vnode)) {
      	//调用update回调以及update钩子
        for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
        if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
		//判断新节点是否有文本
    if (isUndef(vnode.text)) {
      	//新旧节点都有子节点情况下，如果新旧子节点不相同，那么进行子节点的比较，就是updateChildren方法
        if (isDef(oldCh) && isDef(ch)) {
            if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
        } else if (isDef(ch)) {
          	//只有新节点有子节点
            if (process.env.NODE_ENV !== 'production') {
              	//重复Key检测
                checkDuplicateKeys(ch)
            }
          	//清除旧节点文本
            if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
          	//添加新节点
            addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
        } else if (isDef(oldCh)) {
          	//只有旧节点有子节点，删除旧节点
            removeVnodes(oldCh, 0, oldCh.length - 1)
        } else if (isDef(oldVnode.text)) {
          	//新旧节点都无子节点
            nodeOps.setTextContent(elm, '')
        }
    } else if (oldVnode.text !== vnode.text) {
      	//新节点文本替换旧节点文本
        nodeOps.setTextContent(elm, vnode.text)
    }

    if (isDef(data)) {
        if (isDef(i = data.hook) && isDef(i = i.postpatch)) i(oldVnode, vnode)
    }
}
```

配上流程图会更清晰点

![](https://s2.loli.net/2021/12/15/bqZ4ev1ExcHWMs2.png)

![](https://vue-js.com/learn-vue/assets/img/3.7b0442aa.png)

**子节点的比较更新updateChildren**

新旧节点都有子节点的情况下，这个时候是需要调用updateChildren方法来比较更新子节点的。所以在数据上，新旧节点子节点，就保存为了两个数组。

```javascript
const oldCh = [oldVnode1, oldVnode2,oldVnode3];
const newCh = [newVnode1, newVnode2,newVnode3];
```

子节点更新采用的是**双端比较**的策略，什么是双端比较呢，就是新旧节点比较是通过互相比较首尾元素(存在4种比较)，然后向中间靠拢比较(**newStartIdx,与oldStartIdx递增,newEndIdx与oldEndIdx递减**)的策略。

**比较过程**

![](https://vue-js.com/learn-vue/assets/img/8.e4c85c40.png)

**向中间靠拢**

![](https://vue-js.com/learn-vue/assets/img/15.e9bdf5c1.png)

这里对上面出现的**新前，新后，旧前，旧后**做一下说明

1. 新前，指的是**新节点未处理**的子节点数组中的**第一个**元素，对应到vue源码中的**newStartVnode**
2. 新后，指的是**新节点未处理**的子节点数组中的**最后一个**元素,对应到vue源码中的**newEndVnode**
3. 旧前，指的是**旧节点未处理**的子节点数组中的**第一个**元素,对应到vue源码中的**oldStartVnode**
4. 旧后，指的是**旧节点未处理**的子节点数组中的**最后一个**元素,对应到vue源码中的**oldEndVnode**

**子节点比较过程**

接下来对上面的比较过程以及比较后做的操作做下说明

- **新前与旧前**的比较，如果他们相同，那么进行上面说到的patchVnode更新操作，然后新旧节点各向后一步，进行第二项的比较...直到遇到不同才会换种比较方式

![](https://vue-js.com/learn-vue/assets/img/9.e017b452.png)

```javascript
if (sameVnode(oldStartVnode, newStartVnode)) {
  // 更新子节点
  patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
  // 新旧各向后一步
  oldStartVnode = oldCh[++oldStartIdx]
  newStartVnode = newCh[++newStartIdx]
}
```



- **新后与旧后**的比较，如果他们相同，同样进行pathchVnode更新，然后新旧各向前一步，进行前一项的比较...直到遇到不同，才会换比较方式

![](https://vue-js.com/learn-vue/assets/img/10.cf98adc0.png)

```javascript
if (sameVnode(oldEndVnode, newEndVnode)) {
    //更新子节点
    patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
    // 新旧向前
    oldEndVnode = oldCh[--oldEndIdx]
    newEndVnode = newCh[--newEndIdx]
}
```

- **新后与旧前**的比较，如果它们相同，就进行更新操作，然后将旧前移动到**所有未处理旧节点数组最后面**，使旧前与新后位置保持一致，然后双方一起向中间靠拢，新向前，旧向后。如果不同会继续切换比较方式

![](https://vue-js.com/learn-vue/assets/img/12.bace2f7f.png)

```javascript
if (sameVnode(oldStartVnode, newEndVnode)) {
  patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
  //将旧子节点数组第一个子节点移动插入到最后
  canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
  //旧向后
  oldStartVnode = oldCh[++oldStartIdx]
  //新向前
  newEndVnode = newCh[--newEndIdx]
```

- **新前与旧后**的比较，如果他们相同，就进行更新，然后将旧后移动到**所有未处理旧节点数组最前面**，是旧后与新前位置保持一致，，然后新向后，旧向前，继续向中间靠拢。继续比较剩余的节点。如果不同，就使用传统的循环遍历查找。

![](https://vue-js.com/learn-vue/assets/img/14.18c1c6dd.png)

```javascript
if (sameVnode(oldEndVnode, newStartVnode)) {
  patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
  //将旧后移动插入到最前
  canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
  //旧向前
  oldEndVnode = oldCh[--oldEndIdx]
  //新向后
  newStartVnode = newCh[++newStartIdx]
}
```

- **循环遍历查找**，上面四种都没找到的情况下，会通过key去查找匹配。

进行到这一步对于没有设置key的节点，第一次会通过createKeyToOldIdx建立key与index的映射 {key:index}

```javascript
// 对于没有设置key的节点，第一次会通过createKeyToOldIdx建立key与index的映射 {key:index}
if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
```

然后拿新节点的key与旧节点进行比较，找到key值匹配的节点的位置，这里需要注意的是，如果新节点也没key，那么就会执行findIdxInOld方法，从头到尾遍历匹配旧节点

```javascript
//通过新节点的key,找到新节点在旧节点中所在的位置下标,如果没有设置key，会执行遍历操作寻找
idxInOld = isDef(newStartVnode.key)
  ? oldKeyToIdx[newStartVnode.key]
  : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)

//findIdxInOld方法
function findIdxInOld(node, oldCh, start, end) {
  for (let i = start; i < end; i++) {
    const c = oldCh[i]
    //找到相同节点下标
    if (isDef(c) && sameVnode(node, c)) return i
  }
}
```

如果通过上面的方法，依旧没找到新节点与旧节点匹配的下标，那就说明这个节点是新节点，那就执行新增的操作。

```javascript
//如果新节点无法在旧节点中找到自己的位置下标,说明是新元素，执行新增操作
if (isUndef(idxInOld)) {
  createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
}
```

如果找到了，那么说明在旧节点中找到了key值一样，或者节点和key都一样的旧节点。如果节点一样，那么在patchVnode之后，需要将旧节点**移动到所有未处理节点之前**，对于key一样，元素不同的节点，将其认为是新节点，执行新增操作。执行完成后，新节点向后一步。

![](https://vue-js.com/learn-vue/assets/img/6.b9621b4d.png)

```javascript
//如果新节点无法在旧节点中找到自己的位置下标,说明是新元素，执行新增操作
if (isUndef(idxInOld)) {
  // 新增元素
  createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
} else {
  // 在旧节点中找到了key值一样的节点
  vnodeToMove = oldCh[idxInOld]
  if (sameVnode(vnodeToMove, newStartVnode)) {
    // 相同子节点更新操作
    patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
    // 更新完将旧节点赋值undefined
    oldCh[idxInOld] = undefined
    //将旧节点移动到所有未处理节点之前
    canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
  } else {
    // 如果是相同的key，不同的元素，当做新节点，执行创建操作
    createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
  }
}
//新节点向后
newStartVnode = newCh[++newStartIdx]
```

当完成对旧节点的遍历，但是新节点还没完成遍历,那就说明后续的都是新增节点，执行新增操作，如果完成对新节点遍历，旧节点还没完成遍历，那么说明旧节点出现冗余节点，执行删除操作。

```javascript
//完成对旧节点的遍历，但是新节点还没完成遍历，
if (oldStartIdx > oldEndIdx) {
  refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
  // 新增节点
  addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
} else if (newStartIdx > newEndIdx) {
  // 发现多余的旧节点，执行删除操作
  removeVnodes(oldCh, oldStartIdx, oldEndIdx)
}
```

**子节点比较总结**

上面就是子节点比较更新的一个完整过程，这是完整的逻辑代码

```javascript
function updateChildren(parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0] //旧前
    let oldEndVnode = oldCh[oldEndIdx] //旧后
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0] //新前
    let newEndVnode = newCh[newEndIdx] //新后
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    // removeOnly is a special flag used only by <transition-group>
    // to ensure removed elements stay in correct relative positions
    // during leaving transitions
    const canMove = !removeOnly

    if (process.env.NODE_ENV !== 'production') {
        checkDuplicateKeys(newCh)
    }

    //双端比较遍历
    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
        if (isUndef(oldStartVnode)) {
            //旧前向后移动
            oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
        } else if (isUndef(oldEndVnode)) {
            // 旧后向前移动
            oldEndVnode = oldCh[--oldEndIdx]
        } else if (sameVnode(oldStartVnode, newStartVnode)) {
            //新前与旧前
            //更新子节点
            patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
                // 新旧各向后一步
            oldStartVnode = oldCh[++oldStartIdx]
            newStartVnode = newCh[++newStartIdx]
        } else if (sameVnode(oldEndVnode, newEndVnode)) {
            //新后与旧后
            //更新子节点
            patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
                //新旧各向前一步
            oldEndVnode = oldCh[--oldEndIdx]
            newEndVnode = newCh[--newEndIdx]
        } else if (sameVnode(oldStartVnode, newEndVnode)) {
            // 新后与旧前
            //更新子节点
            patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
                //将旧前移动插入到最后
            canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
                //新向前，旧向后
            oldStartVnode = oldCh[++oldStartIdx]
            newEndVnode = newCh[--newEndIdx]
        } else if (sameVnode(oldEndVnode, newStartVnode)) {
            // 新前与旧后
            patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)

            //将旧后移动插入到最前
            canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)

            //新向后，旧向前
            oldEndVnode = oldCh[--oldEndIdx]
            newStartVnode = newCh[++newStartIdx]
        } else {
            // 对于没有设置key的节点，第一次会通过createKeyToOldIdx建立key与index的映射 {key:index}
            if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)

            //通过新节点的key,找到新节点在旧节点中所在的位置下标,如果没有设置key，会执行遍历操作寻找
            idxInOld = isDef(newStartVnode.key) ?
                oldKeyToIdx[newStartVnode.key] :
                findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)

            //如果新节点无法在旧节点中找到自己的位置下标,说明是新元素，执行新增操作
            if (isUndef(idxInOld)) {
                // 新增元素
                createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
            } else {
                // 在旧节点中找到了key值一样的节点
                vnodeToMove = oldCh[idxInOld]
                if (sameVnode(vnodeToMove, newStartVnode)) {
                    // 相同子节点更新操作
                    patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
                        // 更新完将旧节点赋值undefined
                    oldCh[idxInOld] = undefined
                        //将旧节点移动到所有未处理节点之前
                    canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
                } else {
                    // 如果是相同的key，不同的元素，当做新节点，执行创建操作
                    createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
                }
            }
            //新节点向后一步
            newStartVnode = newCh[++newStartIdx]
        }
    }

    //完成对旧节点的遍历，但是新节点还没完成遍历，
    if (oldStartIdx > oldEndIdx) {
        refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
            // 新增节点
        addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
        // 发现多余的旧节点，执行删除操作
        removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
}
```

### 参考资料

1. [VirtualDOM与diff](https://github.com/answershuto/learnVue/blob/master/docs/VirtualDOM%E4%B8%8Ediff(Vue%E5%AE%9E%E7%8E%B0).MarkDown)
2. [渲染页面：浏览器的工作原理](https://developer.mozilla.org/zh-CN/docs/Web/Performance/How_browsers_work)
2. [Vue中的DOM-Diff](https://vue-js.com/learn-vue/virtualDOM/patch.html#_1-%E5%89%8D%E8%A8%80)
2. [深入剖析：Vue核心之虚拟DOM](https://juejin.cn/post/6844903895467032589#heading-19)
