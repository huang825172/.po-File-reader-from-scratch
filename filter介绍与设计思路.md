# filter 介绍与设计思路

## （一）filter 设计目标

filter，即 **过滤器**，用于在将文本送入翻译器之前，把其中包含的 **特殊元素** （HTML 标记，转义字符，占位符等）过滤掉，只保留需要翻译的字符串，使文本流更加纯净。

一个包含多种元素的字符串例子如下：

```
apple <href=123 <span>> hello <\\a> world %s string <h3 font-size=\\
```

其中包括了需要翻译的元素：**apple hello world string**

包含了完整、不完整的 HTML 标记：**-<href=123 <span>> <\\\a> <h3 font-size=\\\-**

包含了格式化占位符：**%s**

filter 的工作就是将 **需要、不需要** 翻译的元素分开。

## （二）设计思路



## （三）代码解析