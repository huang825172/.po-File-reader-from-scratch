# .po 文件解析器设计（一）

## （一）RegExp

RegExp, Regular expression, **正则表达式**。是一种字符串表达式，用于表述一个搜索模式，常用于对于字符串的 **搜索、替换** 算法。一个简单的 RegExp 例子：

```
\bhi\b
```

它匹配字符串中的 "hi" 这个单词。\b 字符匹配单词的开头或结尾。

这是文件解析器中会使用的正则表达式，匹配字符串中被双引号包围的部分：

```
(?<=\")(.+?)(?=\")
```

比如使用此表达式匹配字符串 `msgid "hello"` ，结果将输出 `hello`。

更加详细的正则表达式教程可以参考 [正则表达式30分钟入门教程]( https://deerchao.cn/tutorials/regex/regex.htm )

## （二）文件数据结构设计

由于 .po 文件由多个 **entry** 组成，用于存储文件内容的数据结构也将采用类似的结构，以块为单位，文件的每一个 **entry** 存储在一个块中。块中不仅仅包含待翻译的语句，还要包含此 **entry** 中的注释等额外信息。

所以相应的，在实现解析器时，需要设计以下的数据结构：

| 数据结构 | 存储内容                          | 数据成员             |
| -------- | --------------------------------- | -------------------- |
| po_block | 文件 entry 块待翻译文本及额外内容 | msgid, msgstr, extra |
|po_file|整个文件所有块的信息|blocks|

## （三）代码解析（一）

现在开始，我们来一步步实现 .po 文件的读取和解析。

### 1. 简单的例子

一切的复杂工程都从简单的编码开始，在第一段 python 代码中，我们将仅仅实现读取一个 .po 文件，并将其原封不动的写入另一个 .po 文件：

[1/main.py](https://github.com/huang825172/.po-File-reader-from-scratch-source/blob/master/1/main.py )

```python
infilePath = "../po/ultimate-member-zh_CN.po"
outfilePath = "../out/out_ultimate-member_zh_CN.po"
with \
        open(infilePath, 'r', encoding="UTF-8") as inf, \
        open(outfilePath, 'w', encoding='UTF-8') as outf:
    for line in inf.readlines():
        outf.write(line)
```

代码的前两行定义了两个字符串 **infilePath, outfilePath**，存储输入、输出文件的存储路径。然后，通过 **with open ...** 语句，打开了输入和输出文件。文件打开后，使用 **readlines()** 方法一次将文件内容按行读出，存储在一个列表中，并进行遍历，将每一行写入输出文件。在程序执行离开 **with open ...** 语句后，打开的文件会自动关闭，所以无需手动进行管理。

### 2. 读出 msgid, msgstr 行

既然已经可以读取文件的内容，现在应该着手提取其中有用的部分了。这个版本的代码中，我们将识别出包含 **msgid, msgstr** 内容的文件行：

[2/main.py](https://github.com/huang825172/.po-File-reader-from-scratch-source/blob/master/2/main.py )

```python
infilePath = "../po/ultimate-member-zh_CN.po"
outfilePath = "../out/out_ultimate-member_zh_CN.po"
with \
        open(infilePath, 'r', encoding="UTF-8") as in_f, \
        open(outfilePath, 'w', encoding='UTF-8') as out_f:
    file_lines = in_f.readlines()
    idx = 0
    while idx < len(file_lines):
        line = file_lines[idx]
        if line.find("msgid") == 0:
            while line.find("msgstr") != 0:
                idx += 1
                out_f.write(line)
                line = file_lines[idx]
            line = file_lines[idx]
            out_f.write(line)
            while idx < len(file_lines) - 1 and len(line.replace('\n', '')) > 0:
                idx += 1
                line = file_lines[idx]
                out_f.write(line)
        idx += 1
```

在打开文件后，这里使用 **file_lines** 来存储读取出的字符串列表，列表中的每一项存储着文件内容的每一行。接下来，定义下标变量 **idx** 并初始化为零。接下来的 **while** 循环将不断执行，直到 **idx** 值达到 **file_lines** 下标上限。

在每一次循环中，使 **line** 变量取值为 **file_lines[idx]** ，即对应下标的文件行。如果此行以 **"msgid"** 开头，代表着一个 **entry** 的开始。接下来，在没有找到以 **"msgstr"** 开头的行之前，所有行都属于此 **entry** 的 **"msgid"** 。

在找到 **"msgstr"** 后，直到出现一个 **空行或者文件末尾**， 都属于此 **entry**  的 **"msgstr"** 。

将这些行写入输出文件，就完成了有用部份的提取。

### 3.提取字符串内容

既然已经提取出了包含 **msgid** 与 **msgstr**  的行，接下来就要从行中提取出字符串内容。

[3/main.py](https://github.com/huang825172/.po-File-reader-from-scratch-source/blob/master/3/main.py)

```python
infilePath = "../po/ultimate-member-zh_CN.po"
outfilePath = "../out/out_ultimate-member_zh_CN.mid"
with \
        open(infilePath, 'r', encoding="UTF-8") as in_f, \
        open(outfilePath, 'w', encoding='UTF-8') as out_f:
    file_lines = in_f.readlines()
    file_entries = []
    idx = 0
    while idx < len(file_lines):
        line = file_lines[idx]
        if line.find("msgid") == 0:
            msgid = []
            msgstr = []
            while line.find("msgstr") != 0:
                idx += 1
                if len(line.replace('\n', '')) > 0:
                    msgid.append(line.replace('\n', '').replace('msgid','').strip())
                line = file_lines[idx]
            line = file_lines[idx]
            if len(line.replace('\n', '')) > 0:
                msgstr.append(line.replace('\n', '').replace('msgstr','').strip())
            while idx < len(file_lines) - 1 and len(line.replace('\n', '')) > 0:
                idx += 1
                line = file_lines[idx]
                if len(line.replace('\n', '')) > 0:
                    msgstr.append(line.replace('\n', '').replace('msgstr','').strip())
            file_entries.append((msgid, msgstr))
        idx += 1
    for entry in file_entries:
        out_f.write(str(entry) + '\n')
```

在上一段代码的基础上，这一次进行了更多的过滤操作。使用 **replace** 函数，删除掉字符串中的 **msgid** 和 **msgstr** 部分，使提取出的字符串更加干净。还替换掉了 **\n** **\r** 换行符，并使用 **strip()** 函数清除了两端空白字符。至此，我们已经提取出了包括双引号在内的文本部分，输出看起来是这个样子：

```
--------- snip ----------
(['" users..."'], ['""'])
(['"%s"'], ['""'])
(['"%s - Upgrade Process"'], ['""'])
(['"%s ago"'], ['""'])
(['"%s at %s"'], ['""'])
(['"%s comments"'], ['""'])
(['"%s does not send any user data outside of your site by default."'], ['""'])
(['"%s hr"', '_plural "%s hrs"'], ['[0] ""'])
--------- snip ----------
```

在接下来的教程中，我们将对字符串进行进一步处理，**去掉包围的双引号**，并对于其中 **不能够翻译的部分** （特殊符号，零碎的 HTML 标记等）进行处理，得到能够用于调用翻译 API 的纯净文本。