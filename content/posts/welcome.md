---
title: "Welcome To NyanC's Blog"
date: 2021-03-16T22:07:42+08:00
draft: false
tags: ["blog", "markdown"]
summary: "welcome"
---
---
测试新博客,随便写写,旧博客的文章不迁移了.:dog:

Markdown是一种轻量级标记语言,创始人为John Gruber. Markdown对于图片,图表和数学有相应的支持.

### 标题

Markdown使用 **`#`** 符号表示标题标签.不同数量表示不同级别的标题.
```
# 一级标题

## 二级标题

### 三级标题
```

### 样式

Markdown可使用特殊调节语法字体样式,包括倾斜,加粗等。
```
`代码`  **加粗** *倾斜*  ==标记== ~~删除~~ > 引用 >> 引用嵌套
```

### 图片と链接
```
![Description](PicturenURL)
[Description](LinkURL)
```

### 代码块
代码块用两个 **\`\`\`** 或四个空格或一个制表符表示,代码块中间的文字将不使用Markdown语法解析.采用Highlight则需要在第一组三个点之后加上需要高亮的语法.
``` shell
    #!/bin/bash
    echo hello
```


### 分割线
三个以上的符号
```
--- *** +++ ___
```

### 列表と表单
使用数字加半角句点创建有序列表,使用特殊符号创建无序表单.

```
1. Item1
2. Item2
4. Item3
```
```
+ Item
* Item
- Item
    - Indented item
```
也可以使用`[ ]` `[x]`语法添加勾选样式.
```
- [ ] Item
- [x] Item
```
- [x] Item
- [ ] Item

要添加表单,使用三个以上的连字符(---)创建标题,使用(|)分隔列.

```
|Title1|Title2|
|------|------|
|Item1 |Item2 |
```
|Title1|Title2|
|------:|------:|
|Item1 |Item2 |

### Emoji
``` 
Next Time \:joy\:
```
Next Time :joy:

![](/images/NextTime.png)



