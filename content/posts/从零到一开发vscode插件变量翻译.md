---
title: "从零到一开发vscode插件变量翻译"
date: 2022-01-03T14:48:42+08:00
draft: false
---

## 从零到一开发vscode插件变量翻译

需求的起因是英语渣在开发的过程中经常遇到一个变量知道中文叫啥，但是英文单词可能就忘了，或者不知道，这个时候，我之前做法是打开浏览器，打开谷歌翻译，输入中文，复制英文，然后切回vscode，粘贴结果。实在是太麻烦了，年轻的时候还好，记性好，英文单词大部分都能记住，但随着年纪越来越大，头发越来越少，记性也是越来越差，上面的步骤重复的次数也越来越多，所以痛定思痛，就开发了这款插件。因为自己也是这几天从零开始学习的插件开发，所以本文完全的记录了一个小白开发的插件开发之路，内容主要是实战类的介绍，主要从四个方面介绍来完整的展示整个插件从功能设计到发布的完整历程。

1. 功能设计
2. 环境搭建
3. 插件功能开发
4. 插件打包发布

![](https://s2.loli.net/2021/12/30/NotcCIyDVFAQ4dp.gif)



### 功能设计

功能主要是两个功能，中译英，其他语言翻译成中文

1. 将中文变量替换为翻译后的英文变量，多个单词需要自动驼峰，解决的场景就是日常开发经常碰到的“英语卡壳”
2. 划词翻译，自动将各种语言翻译成中文，这解决的场景是有时候看国外项目源代码的注释碰到不会的单词不知道啥意思影响效率



### 环境搭建

上手的第一步，环境搭建

1. 安装脚手架, **yo** 与 **generator-code**,这两个工具可以帮助我们快速构建项目，详情可见 [Github](https://github.com/Microsoft/vscode-generator-code)

   ```javascript
   //安装
   yarn global add yo generator-code
   ```

2. 安装vsce,vsce可用来将开发的代码打包成.vsix后缀的文件，方便上传至微软插件商店或者本地安装

   ```javascript
   yarn global add vsce
   ```

3. 生成并初始化项目,初始化信息根据自己情况填写

   ```javascript
   //初始化生成项目
   yo code
   ```

   到这一步后，选择直接打开，**Open with code**

   ![image-20211230215507341](https://s2.loli.net/2021/12/30/ftdJaMN7WlgCEOq.png)打开后会自动建立一个工作区，并生成这些文件，可根据自己需要对文件进行删减，完成这步后，我们可以直接进行开发与调试了

   ![image-20211230215823987](https://s2.loli.net/2021/12/30/hkRxvGJ2nwg8Eaj.png)

   **如何进行调试？**

   去**运行与调试**面板点击**Run Extention**,或者快捷键**F5**,mac可以直接点击触控栏的调试按钮

   ![image-20211230220226620](https://s2.loli.net/2021/12/30/X6UAOJH1L9QSaRt.png)

   打开后会弹出一个新的vscode窗口，这个新的窗口就是你的测试环境(**扩展开发宿主**)，你做的插件功能就是在这个新的窗口测试，打印的消息在前一个窗口的**调试控制台**中,比如自带的例子，在我们新窗口 **cmd/ctrl+shift+p**后输入**Hello world**后会在前一个窗口的控制台打印一些信息

   ![image-20211230221707832](https://s2.loli.net/2021/12/30/rP4e3ySlAVsw7MJ.png)

   到这里，开发准备环境就准备好了，下一步就是开始正式的插件功能开发



### 插件功能开发

插件开发中，有两个重要的文件，一个是 **package.json**,一个是 **extention.js**

#### 重要文件说明

##### package.json

![image-20211230222536296](https://s2.loli.net/2021/12/30/vedVAh8xyJmTwN2.png)

- **activationEvents**用来注册激活事件，用来表明什么情况下会激活extention.js中的active函数。常见的有**onLanguage**,**onCommand**...更多信息可查看[vscode文档activationEvents](https://code.visualstudio.com/api/references/activation-events)

- **main**表示插件的主入口

- **contributes**用来**注册命令(commands)**,**绑定快捷键(keybindings)**,**配置设置项(configuration)**等等,更多可配置项可看[文档](https://code.visualstudio.com/api/references/contribution-points)

##### extention.js

extention.js主要作用是作为插件功能的实现点，通过active,deactive函数，以及vscode提供的api以及一些事件钩子来完成插件功能的开发

#### 实现翻译功能

翻译这里主要是使用了两个服务，谷歌和百度翻译。

1. 谷歌翻译参考了别人的做法，使用**google-translate-token**获取到token，然后构造请求url,再处理返回的body,拿到返回结果。这里还有一个没搞懂的地方就是请求url的生成很迷，不知道这一块是啥意思。

   ```javascript
   const qs = require('querystring');
   const got = require('got');
   const safeEval = require('safe-eval');
   const googleToken = require('google-translate-token');
   const languages = require('../utils/languages.js');
   const config = require('../config/index.js');

   // 获取请求url
   async function getRequestUrl(text, opts) {
       let token = await googleToken.get(text);
       const data = {
           client: 'gtx',
           sl: opts.from,
           tl: opts.to,
           hl: opts.to,
           dt: ['at', 'bd', 'ex', 'ld', 'md', 'qca', 'rw', 'rm', 'ss', 't'],
           ie: 'UTF-8',
           oe: 'UTF-8',
           otf: 1,
           ssel: 0,
           tsel: 0,
           kc: 7,
           q: text
       };
       data[token.name] = token.value;
       const requestUrl = `${config.googleBaseUrl}${qs.stringify(data)}`;
       return requestUrl;
   }

   //处理返回的body
   async function handleBody(url, opts) {
       const result = await got(url);
       let resultObj = {
           text: '',
           from: {
               language: {
                   didYouMean: false,
                   iso: ''
               },
               text: {
                   autoCorrected: false,
                   value: '',
                   didYouMean: false
               }
           },
           raw: ''
       };

       if (opts.raw) {
           resultObj.raw = result.body;
       }
       const body = safeEval(result.body);

       // console.log('body', body);
       body[0].forEach(function(obj) {
           if (obj[0]) {
               resultObj.text += obj[0];
           }
       });

       if (body[2] === body[8][0][0]) {
           resultObj.from.language.iso = body[2];
       } else {
           resultObj.from.language.didYouMean = true;
           resultObj.from.language.iso = body[8][0][0];
       }

       if (body[7] && body[7][0]) {
           let str = body[7][0];

           str = str.replace(/<b><i>/g, '[');
           str = str.replace(/<\/i><\/b>/g, ']');

           resultObj.from.text.value = str;

           if (body[7][5] === true) {
               resultObj.from.text.autoCorrected = true;
           } else {
               resultObj.from.text.didYouMean = true;
           }
       }
       return resultObj;
   }

   //翻译
   async function translate(text, opts) {
       opts = opts || {};
       opts.from = opts.from || 'auto';
       opts.to = opts.to || 'en';

       opts.from = languages.getCode(opts.from);
       opts.to = languages.getCode(opts.to);

       try {
           const requestUrl = await getRequestUrl(text, opts);
           const result = await handleBody(requestUrl, opts);
           return result;
       } catch (error) {
           console.log(error);
       }
   }

   // 获取翻译结果
   const getGoogleTransResult = async(originText, ops = {}) => {
       const { from, to } = ops;
       try {
           const result = await translate(originText, { from: from || config.defaultFrom, to: to || defaultTo });
           console.log('谷歌翻译结果', result.text);
           return result;
       } catch (error) {
           console.log(error);
           console.log('翻译失败');
       }
   }

   module.exports = getGoogleTransResult;
   ```



2. 百度翻译，百度翻译的比较简单，申请服务，获得appid和key，然后构造请求url直接请求就行，不知道如何申请的，可查看我之前的一篇文章[Electron+Vue从零开始打造一个本地文件翻译器](https://juejin.cn/post/6899581622471884813)进行申请

   ```javascript
   const md5 = require("md5");
   const axios = require("axios");
   const config = require('../config/index.js');
   axios.defaults.withCredentials = true;
   axios.defaults.crossDomain = true;
   axios.defaults.headers.post["Content-Type"] =
       "application/x-www-form-urlencoded";

   // 百度翻译
   async function getBaiduTransResult(text = "", opt = {}) {
       const { from, to, appid, key } = opt;
       try {
           const q = text;
           const salt = parseInt(Math.random() * 1000000000);
           let str = `${appid}${q}${salt}${key}`;
           const sign = md5(str);
           const query = encodeURI(q);
           const params = `q=${query}&from=${from}&to=${to}&appid=${appid}&salt=${salt}&sign=${sign}`;
           const url = `${config.baiduBaseUrl}${params}`;
           console.log(url);
           const res = await axios.get(url);
           console.log('百度翻译结果', res.data.trans_result[0]);
           return res.data.trans_result[0];
       } catch (error) {
           console.log({ error });
       }
   }

   module.exports = getBaiduTransResult;
   ```

#### 获取选中的文本

使用事件钩子**onDidChangeTextEditorSelection**，获取选中的文本

```javascript
    onDidChangeTextEditorSelection(({ textEditor, selections }) => {
        text = textEditor.document.getText(selections[0]);
    })
```

#### 配置项的获取更新

通过**vscode.workspace.getConfiguration**获取到工作区的配置项，然后通过事件钩子**onDidChangeConfiguration**监听配置项的变动。

**获取更新配置项**

```javascript
const { getConfiguration } = vscode.workspace;
const config = getConfiguration();

//注意get里面的参数其实就是package.json配置项里面的contributes.configuration.properties.xxx
const isCopy = config.get(IS_COPY);
const isReplace = config.get(IS_REPLACE);
const isHump = config.get(IS_HUMP);
const service = config.get(SERVICE);
const baiduAppid = config.get(BAIDU_APPID);
const baiduKey = config.get(BAIDU_KEY);

//更新使用update方法，第三个参数为true代表应用到全局
config.update(SERVICE, selectedItem, true);
```

**监听配置项的变动**

```javascript
const { getConfiguration, onDidChangeConfiguration } = vscode.workspace;
const config = getConfiguration();

//监听变动
const disposeConfig = onDidChangeConfiguration(() => {
  config = getConfiguration();
})

```

**监听个别配置项的变动**

```javascript
const disposeConfig = onDidChangeConfiguration((e) => {
    if (e && e.affectsConfiguration(BAIDU_KEY)) {
        //干些什么
    }
})
```

#### 获取当前打开的编辑器对象

**vscode.window.activeTextEditor**代表当前打开的编辑器，如果切换标签页，而没有设置监听，那么这个这个对象不会自动更新，所以需要使用**onDidChangeActiveTextEditor**来监听，并替换之前的编辑器对象

```javascript
const { activeTextEditor, onDidChangeActiveTextEditor } = vscode.window;
let active = activeTextEditor;
const edit = onDidChangeActiveTextEditor((textEditor) => {
  console.log('activeEditor改变了');
  //更换打开的编辑器对象
  if (textEditor) {
      active = textEditor;
  }
})
```

#### 划词翻译悬浮提示

通过**vscode.languages.registerHoverProvider**注册一个Hover，然后通过**activeTextEditor**拿到选中的词语进行翻译，然后再通过**new vscode.Hover**将翻译结果悬浮提示

```javascript
// 划词翻译检测
const disposeHover = vscode.languages.registerHoverProvider("*", {
    async provideHover(document, position, token) {
        const service = config.get(SERVICE);
        const baiduAppid = config.get(BAIDU_APPID);
        const baiduKey = config.get(BAIDU_KEY);

        let response, responseText;
        const selected = document.getText(active.selection);
        // 谷歌翻译
        if (service === 'google') {
            response = await getGoogleTransResult(selected, { from: 'auto', to: 'zh-cn' });
            responseText = response.text;
        }

        // 百度翻译
        if (service === 'baidu') {
            response = await getBaiduTransResult(selected, { from: "auto", to: "zh", appid: baiduAppid, key: baiduKey });
            responseText = response.dst;
        }
        // 悬浮提示
        return new vscode.Hover(`${responseText}`);
    }
})
```

#### 替换选中的文本

获取到**activeTextEditor**,调用他的**edit**方法,然后使用回调中的**replace**

```javascript
//是否替换原文
if (isReplace) {
  let selectedItem = active.selection;
  active.edit(editBuilder => {
    editBuilder.replace(selectedItem, result)
  })
}
```

#### 复制到剪贴板

使用**vscode.env.clipboard.writeText;**

```javascript
// 是否复制翻译结果
if (isCopy) {
  vscode.env.clipboard.writeText(result);
}
```

#### 驼峰处理

```javascript
function toHump(str) {
    if (!str) {
        return
    }
    const strArray = str.split(' ');
    const firstLetter = [strArray.shift()];
    const newArray = strArray.map(item => {
        return `${item.substring(0,1).toUpperCase()}${item.substring(1)}`;
    })
    const result = firstLetter.concat(newArray).join('');
    return result;
}

module.exports = toHump;
```

#### 快捷键绑定

通过**vscode.commands.registerCommand**注册绑定之前package.json中设置的keybindings,需要注意的是registerCommand的第一个参数需要与keybindings的command保持一致才能绑定

```javascript
registerCommand('translateVariable.toEN', async() => {
  //do something
})


//package.json
"keybindings": [{
  "key": "ctrl+t",
  "mac": "cmd+t",
  "when": "editorTextFocus",
  "command": "translateVariable.toEN"
}],
```



### 插件打包发布

#### 打包

```javascript
vsce package
```

打包后会在目录下生成.vsix后缀的插件

#### 发布

插件发布主要是把打包的vsix后缀插件，传入微软vscode插件商店，当然也能本地安装使用。

**传入商店**

发布到线上需要到[微软插件商店管理页面](https://marketplace.visualstudio.com/manage/createpublisher),创建发布者信息，如果没有微软账号，需要去申请。

![image-20211229224224632](https://s2.loli.net/2021/12/29/jqc7iZBuKoMDPW8.png)

创建完成后，选择发布到vscode商店

![image-20211229224338826](https://s2.loli.net/2021/12/29/mfu7HQlzTIR3cp9.png)

**本地安装**

本地是可以直接安装.vsix后缀插件的，找到插件菜单

![image-20211229224545688](https://s2.loli.net/2021/12/29/cnoKPWj5Ypq2URk.png)

选择**从VSIX安装**,安装上面打包的插件就好了

![image-20211229224658287](https://s2.loli.net/2021/12/29/3XdploSMuOJ65TF.png)

### 最后

vscode的中文资料有点少，这次开发大多数时间都在看英文文档，以及上外网寻找资料，真的英语太重要了，后面得多学点英语了，希望后面我使用自己做的这个插件的次数会越来越少，项目已开源，[使用说明与源码传送门](https://github.com/Kerinlin/translate-variable)
