---
title: "Welcome To NyanC's Blog"
date: 2021-03-16T22:07:42+08:00
draft: false
tags: ["blog", "markdown"]
summary: "The summary content"
---
---

&ensp;&ensp;Markdown是一种轻量级标记语言,创始人为John Gruber. Markdown对于图片,图表和数学有相应的支持.

### 标题

&ensp;&ensp;Markdown使用 **`#`** 符号表示标题标签.不同数量表示不同级别的标题.
```
# 一级标题

## 二级标题

### 三级标题
```

### 样式

&ensp;&ensp;Markdown可使用特殊调节语法字体样式,包括倾斜,加粗等。
```
`突出`  **加粗** *倾斜*  ==标记== --删除-- > 引用 
```

### 图片と链接
```
![Description](PicturenURL)
[Description](LinkURL)
```

### 文本
文本框用两个 **```** 表示,文本框中间的文字将不使用Markdown语法解析.采用Highlight则需要在第一组三个点之后加上需要高亮的语法.
```
` ` `c
int main() {
    puts("hello");
}
` ` `
` ` `bash
#!/bin/bash
echo hello
` ` `
```

### 分割线
三个以上的符号
```
--- *** +++ ___
```