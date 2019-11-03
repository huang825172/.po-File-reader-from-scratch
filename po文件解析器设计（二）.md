# .po 文件解析器设计（二）

## （一）代码解析（二）

[4/main.py](https://github.com/huang825172/.po-File-reader-from-scratch-source/blob/master/4/main.py )

```python
import re

infilePath = "../po/ultimate-member-zh_CN.po"
outfilePath = "../out/out_ultimate-member_zh_CN.mid"


def strip_rn(line_str):
    return line_str.replace('\n', '').replace('\r', '').strip()


def strip_line(line_str):
    pattern = re.compile('(?<=\")(.+?)(?=\")')
    result = pattern.search(line_str)
    if result is None:
        return ""
    else:
        return result.group().strip()


try:
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
                    if len(strip_rn(line)) > 0:
                        msgid.append(strip_line(line))
                    line = file_lines[idx]
                line = file_lines[idx]
                if len(strip_rn(line)) > 0:
                    msgstr.append(strip_line(line))
                while idx < len(file_lines) - 1 and len(strip_rn(line)) > 0:
                    idx += 1
                    line = file_lines[idx]
                    if len(strip_rn(line)) > 0:
                        msgstr.append(strip_line(line))
                file_entries.append((msgid, msgstr))
            idx += 1
        for entry in file_entries:
            out_f.write(str(entry) + '\n')
except Exception as e:
    print(e)
```

首先，这个程序中将上次用于清除换行符、空白字符、msgid、msgstr 的代码 **独立成为函数** ，提高了代码的可读性。

第二，这个程序中使用了 **try except** 语句块进行异常捕捉，保证程序在遇到异常时能够正常退出。

第三，为了提取出双引号中的文字，这一版的代码引入了 **re** 正则表达式处理库，并编译了 **(?<=\")(.+?)(?=\")** 模式用于匹配双引号中的文本（这里只需要了解其作用，如果感兴趣原理可以到前面章节提供的链接学习）。使用 **search** 函数，将字符串中被双引号包裹的部分提取出来，这一次的输出是这样：

```
--------- snip ----------
(['%s of %d'], [''])
(['%s provides you with forms for user registration, login and profiles.'], [''])
(['%s year old'], [''])
(['%s years old'], [''])
(['&mdash; No role for Ultimate Member &mdash;'], [''])
(['( 12-hr format )'], [''])
(['( 24-hr format )'], [''])
(['(empty)'], [''])
(['1 comment'], [''])
(['10 stars rating system'], [''])
(['5  stars rating system'], [''])
--------- snip ----------
```

[5/main.py](https://github.com/huang825172/.po-File-reader-from-scratch-source/blob/master/5/main.py )

这一版本的代码没有太多变化。

改为使用空行，将文件分割为多个 **block**，然后分别读取。这样处理的好处是能够将每一个语句块对应的 **注释信息和待翻译语句存储在一起** ，方便后续写入文件。

有兴趣可以自己研究一下代码。