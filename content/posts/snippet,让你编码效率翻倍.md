---
title: "snippet,让你编码效率翻倍"
date: 2021-03-25T11:53:25+08:00
draft: false
---

## 为什么谈到Snippet ##
今天下午在用vscode做小程序的时候，发现很不方便，因为商店里提供的代码片段极为有限，而且平时几乎每天都需要用到代码片段，所以就在思考他们是怎么做到给别人提供代码的，我可以自定义代码片段吗。然后查了下，果然，这在vscode里自带的（好像藏得有点深），是可以自定义的，然后在做完自己的任务后捣鼓了下，基本了解了snippet的语法，突然有种打开新世界大门的感觉。做个记录，上菜了

----------

## 如何打开snippet配置 ##
这里以vscode为例，其他编辑器大概也差不多。在vscode中快捷键「**Ctrl + Shift + P**」打开命令窗口，然后输入**snippet**,选择 **[配置用户代码片段]**，点击后，就可以愉快的进行片段的编写了


![clipboard.png](https://image-static.segmentfault.com/231/434/2314342582-5b99eb2c6de41_fix732)



![clipboard.png](https://image-static.segmentfault.com/128/771/1287718887-5b99eb3f0915c_fix732)


----------


## Snippet怎么用 ##

### 先上一个Demo ###

```
"html template": {
    "prefix": "ht",
    "body": [
      "<!DOCTYPE html>",
      "<html lang=\"en\">",
      "<head>",
      "  <meta charset=\"UTF-8\">",
      "  <title>${1:$CURRENT_DATE}</title>",
      "</head>",
      "<body>",
			" <div class=\"${2|container,wrapper|}\">",
			"   ${3}",
			" </div>",
      "</body>",
      "</html>",
    ],
    "description": "create a html frame"
  }
```
效果是这样滴
![clipboard.png](https://image-static.segmentfault.com/609/331/609331190-5b99dcaaadc99_fix732)

### 基础结构 ###

![clipboard.png](https://image-static.segmentfault.com/709/183/709183500-5b99c925de998_fix732)

-    片段名字
-    prefix（前缀，输入的触发条件，比如上面例子中当我输入ht后，就能tab出来片段）
-    body（主体部分，在里面根据语法定义自己需要的代码片段）
-    description（说明，片段的具体描述）

### 基础语法 ###

 - 每个逗号代表一整行的结束，双引号需要用转义字符 \
 - $number表示光标跳转的顺序，比如$1表示光标首次需要跳转的位置，相同序号的会在一起，另外$0表示最终光标位置
 - 变量，在未赋值的情况下提供默认值，这里提供一些变量
```
    TM_SELECTED_TEXT：当前选定的文本或空字符串；
    TM_CURRENT_LINE：当前行的内容；
    TM_CURRENT_WORD：光标所处单词或空字符串
    TM_LINE_INDEX：行号（从零开始）；
    TM_LINE_NUMBER：行号（从一开始）；
    TM_FILENAME：当前文档的文件名；
    TM_FILENAME_BASE：当前文档的文件名（不含后缀名）；
    TM_DIRECTORY：当前文档所在目录；
    TM_FILEPATH：当前文档的完整文件路径；
    CLIPBOARD：当前剪贴板中内容。
    时间相关
    CURRENT_YEAR: 当前年份；
    CURRENT_YEAR_SHORT: 当前年份的后两位；
    CURRENT_MONTH: 格式化为两位数字的当前月份，如 02；
    CURRENT_MONTH_NAME: 当前月份的全称，如 July；
    CURRENT_MONTH_NAME_SHORT: 当前月份的简称，如 Jul；
    CURRENT_DATE: 当天月份第几天；
    CURRENT_DAY_NAME: 当天周几，如 Monday；
    CURRENT_DAY_NAME_SHORT: 当天周几的简称，如 Mon；
    CURRENT_HOUR: 当前小时（24 小时制）；
    CURRENT_MINUTE: 当前分钟；
    CURRENT_SECOND: 当前秒数。

```
 - 可选项，当光标到该处的时候弹出一些可选择项，使用 | ，| 后面是自己提供的可选项 我这里是提供了两个值，值之间使用逗号进行分隔

![clipboard.png](https://image-static.segmentfault.com/926/289/926289316-5b99e38d46001_fix732)

 - body的高级语法，可以参考[这里][1]，写的很详细

----------
## 最后 ##
效果

![clipboard.png](https://image-static.segmentfault.com/167/703/1677035697-5b99f2b94bd0d_fix732)

最后附上把自己的snippet放到market上的教程，使劲戳[这里][2]


  [1]: https://blog.csdn.net/maokelong95/article/details/54379046#34-body-%E9%AB%98%E7%BA%A7%E8%AF%AD%E6%B3%95
  [2]: https://blog.csdn.net/crper/article/details/78637080