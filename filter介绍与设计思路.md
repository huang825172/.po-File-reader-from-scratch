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

经过统计，在网站翻译工作中包含的特殊符号有如下几种：

+ HTML 标记 *e.g.* **"<a href=>"**
+ 由转义引号包含的部分 *e.g.* **"\\\"text\\\""**
+ 由方括号包含的部分 *e.g.* **"[text]"**
+ 由 **%** 开头的格式化占位符 *e.g.* **"%1$s %s"**
+ 由 **&# ;** 包含的 HTML 实体 *e.g.* **"\&#8211;"**

上述符号都不应该被翻译，而是被原样保留。

## （二）设计思路

由于 **filter** 需要做的工作有两部分：将文本分段、标记需要翻译的文段，我们需要输出这两部分工作的结果。所以这里将 **filter** 的输出设计为两个长度相同的列表：

| ---          | ---                                                          |
| ------------ | ------------------------------------------------------------ |
| segments     | 用于存储文本分段结果的列表，存储一系列字符串。               |
| segments_map | 用于存储翻译信息的列表，长度与 **segments** 相同，如果 **segments_map[index]** 为 0，表示 **segments[index]** 存储的字符串不需要被翻译；若为 1 则需要。 |

这里举一个 filter 工作的例子：

**输入字符串：** "Hello <a> world"

**输出 segments：** ['Hello',  '<a>',  'world']

**输出 segments_map：** [1, 0, 1]

## （三）代码解析

由于实际 filter 代码较为复杂，这里只节选其中几段分析。

首先，是用于 **括号过滤** 的函数。它在 filter 中执行了 **过滤 HTML 标签** 和 **过滤方括号** 的工作。实际上，这个函数可以用于任何类似括号（有左右之分）符号的过滤。其算法设计保证了及时文段中的括号非成对匹配，即存在不完整标记的情况下，也能将其过滤。各段代码的作用已经标注在了注释里：

```python
def _filte_html(self, raw_str):
        sum = 0
        addl = 0
        addr = 0
        for c in raw_str:
            if c == "<":
                sum += 1
            elif c == ">":
                sum -= 1
            if sum < 0:
                addl += -sum
                sum = 0
        addr = sum
        fited_str = "<" * addl + raw_str + ">" * addr
        # 以上进行了括号的补全
        # 通过遍历，计算要将字符串括号全部配对，需要补全的括号数量
        # 并将需要补的括号加在字符串两端
        
        sum = 0
        segs = []
        segs_map = []
        seg = ""
        seg_flag = 1
        for idx in range(len(fited_str)):
            c = fited_str[idx]
            if idx == 0 and c == "<":
                seg_flag = 0
            if c == "<":
                if len(seg) > 0:
                    segs.append(seg)
                    segs_map.append(seg_flag)
                    seg_flag = 0
                    seg = ""
                sum += 1
                seg += c
                if idx == len(fited_str) - 1:
                    if len(seg) > 0:
                        segs.append(seg)
                        segs_map.append(seg_flag)
            elif c == ">":
                sum -= 1
                seg += c
                if idx == len(fited_str) - 1:
                    if len(seg) > 0:
                        segs.append(seg)
                        segs_map.append(seg_flag)
            elif sum == 0 and idx != 0 and c != ">":
                if len(seg) > 0 and seg_flag == 0:
                    segs.append(seg)
                    segs_map.append(seg_flag)
                    seg_flag = 1
                    seg = ""
                seg += c
                if idx == len(fited_str) - 1:
                    if len(seg) > 0:
                        segs.append(seg)
                        segs_map.append(seg_flag)
            elif idx == len(fited_str) - 1:
                if len(seg) > 0:
                    segs.append(seg)
                    segs_map.append(seg_flag)
            else:
                seg += c
        # 这一段进行了文本分段和分类
        # 通过分类讨论，分离出括号包裹的文本，标记为不需要翻译
        # 最终形成 segs 与 segs_map 两个列表
                
        if addl > 0:
            segs[0] = segs[0][addl:]
        if addr > 0:
            segs[-1] = segs[-1][:-addr]
        # 由于一开始在字符串两边补全了括号
        # 这段代码负责去除补全的括号，让字符串恢复原状
            
        return segs, segs_map
```

接下来是一段用于 **提取 HTML 实体** 的代码。类似的代码被用于提取其他可以确定首尾元素的文段（例如引号，格式化占位符等）。各段代码的作用已经标注在了注释里:

```python
def _filte_html_entities(self, segs, segs_map):
    # 由于 filter 函数是链式调用的，这个函数的输入参数是上一个函数输出的 segs 与 segs_map
        seg = ""
        new_segs = []
        new_segs_map = []
        for idx in range(len(segs)):
            if segs_map[idx] == 1:
            # 仅对前一个 filter 函数认为要翻译的部分进行处理
                str = segs[idx]
                str_idx = 0
                while str_idx < len(str):
                    if str[str_idx:str_idx + 2] == '&#':
                    # 如果扫描到起始字符，则开始寻找结束字符
                        if len(seg) > 0:
                            new_segs.append(seg)
                            new_segs_map.append(1)
                            seg = ""
                        fpos = str_idx + 2
                        for fidx in range(str_idx, len(str)):
                            if str[fidx] == ";":
                            # 寻找到结束字符的位置
                                fpos = fidx + 1
                                break
                        new_segs.append(chr(int(str[str_idx:fpos].replace(\
                        	"&#", '').replace(';', ''))))
                        # 由于这里是扫描 HTML 实体标记，直接将其转换为了可读字符
                        new_segs_map.append(1)
                        str_idx += fpos - str_idx
                        # 将截取的段落提取出来，并对 idx 进行更改，跳过已扫描的段落
                    else:
                        seg += str[str_idx]
                        if str_idx == len(str) - 1:
                            new_segs.append(seg)
                            new_segs_map.append(1)
                            seg = ""
                        str_idx += 1
            else:
            # 如果遇到不需要翻译的段落，直接将其加入新的段落列表
                new_segs.append(segs[idx])
                new_segs_map.append(segs_map[idx])
        return new_segs, new_segs_map
```

最后分析一段优化器的代码，这段代码在所有 filter 函数之后运行，对它们的分段结果进行优化，去除被误判需要翻译的项目，代码非常简单，但是有效提高了后续的翻译性能。这里直接贴出代码：

```python
def _optimizer(self, segs, segs_map):
        new_segs_map = []
        for idx in range(len(segs)):
            seg = segs[idx].strip()
            if\
                seg == '.' or\
                seg == '\\' or\
                seg == '':
                new_segs_map.append(0)
            else:
                new_segs_map.append(segs_map[idx])
        return segs, new_segs_map
```

具体的 filter 实现请查阅 [Github 上的代码](https://github.com/huang825172/.po-File-reader-from-scratch-source/blob/master/6/poIO/PoBlockFilters.py )