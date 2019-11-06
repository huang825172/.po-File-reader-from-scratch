# .po 文件格式详解

## （一）GNU gettext 工具

（这一部分内容有所了解即可）

GNU，是“ GNU's Not Unix ”的递归缩写，是一个[自由软件](http://www.gnu.org/philosophy/free-sw.html)操作系统—就是说，它尊重其使用者的自由。**自由软件意味着使用者有运行、复制、发布、研究、修改和改进该软件的自由。**在 GNU 软件包中，gettext 工具旨在为程序的 **i18n**（internationalization 国际化）和 **l10n**（localization 本地化）提供支持。

使用 gettext 工具的一般流程：

+ 英文源代码：使用英文编写程序源代码
+ 备注并标记要翻译语句：在源代码中加入 gettext 相关的声明信息，并标记需要翻译的语句
+ 提取要翻译语句：使用 gettext 工具提取语句，生成 .pot 文件
+ 生成.po文件：基于 .pot 文件生成对应特定语言的 .po 文件
+ **翻译.po文件：对 .po 文件进行翻译**
+ 转化为.mo格式：将 .po 文件转化为 .mo 文件，重新进行程序构建

总而言之，GNU gettext 工具可以 **用于生成 .po 文件**。而对 .po 文件的解析，提取出要翻译的字符串是我们项目的重点。

## （二）.pot 文件和 .po 文件

.pot，Portable Object Template，**可移植对象模板文件**，它是生成.po文件过程中一个跳板。

.pot 与 .po 文件都是纯文本文件格式，文件中包含三种内容：


+ **注释：** 类似代码中的注释，可以编写任意文本内容，不需要被翻译。注释行以 `#` 开头，具体的注释类型由紧随井号的字符决定，即可有如下几种：（该表格稍作了解即可，实际使用时很少区分）
| 起始符号 | 符号解释 | 包含内容 |
| ---- | ---- | ---- |
|"# "|紧跟空格|根据提取指令生成，以及翻译者所编写的注释|
|"#."|紧跟句点|额外注释，从源代码注释生成 |
|"#:"|紧跟冒号|表明待翻译语句的出处，一般标记源代码文件及行数|
|"#,"|紧跟逗号|由编译器生成的格式注释|
|"#\|"|紧跟 \||上一句未翻译文本，较少见|
+ **翻译语句块：** 由成对的 **msgid** 与 **msgstr** 标记组成
  + msgid 后跟一或多个由双引号包围的**待翻译字符串**，直到出现 msgstr
  + msgstr 后跟一个或多个由双引号包围的**翻译字符串**
  + 如果该文段尚未翻译，msgstr 后跟一个空字符串
+ **额外信息：** 文件开头第一个 msgstr 后的双引号包围字符串
  + 这个 msgstr 所对应的 msgid 没有内容
  + 一般描述了翻译人，时间日期等信息
  + 不需要被翻译，可以当作注释对待



**除了 #: 以外的所有注释、额外信息都是可选的**

**即使文件中没有任何这些信息，文件同样是符合规范的**

实际上，.pot 文件与 .po 文件的格式是相同的。.pot 文件是用于生成 .po 文件的模板，其不包含特定语言、作者信息。在利用工具生成 .pot 文件后，可以对其中包含的作者、翻译者、日期等信息进行修改，再生成包含特定语言信息的 .po 文件。

## （三）实例解析

这里通过讲解简单的 .pot 和 .po 文件例子来帮助了解文件格式。下面是一个 .pot 文件：

```po
msgid ""
msgstr ""
"Project-Id-Version: Ultimate Member\n"
"Report-Msgid-Bugs-To: \n"
"POT-Creation-Date: 2019-10-24 08:22+0000\n"
"PO-Revision-Date: 2019-10-24 08:23+0000\n"
"Last-Translator:\n"
"Language-Team:\n"
"Language:\n"
"Plural-Forms: nplurals=1; plural=0;\n"
"MIME-Version: 1.0\n"
"Content-Type: text/plain; charset=UTF-8\n"
"Content-Transfer-Encoding: 8bit\n"
"X-Generator: Loco https://localise.biz/\n"
"X-Loco-Version: 2.2.2; wp-5.0.7"

#: includes/core/class-files.php:1125
msgid " "
msgstr ""

#: includes/admin/core/class-admin-upgrade.php:248
#, php-format
msgid "%s - Upgrade Process"
msgstr ""
```

文件开头的第一个 **msgid, msgstr对** 描述了文件的额外信息，包括创建时间，文件格式等，但在作者相关和语言相关的条目中没有信息。

在第一段最后一行 `...wp-5.0.7"` 之后开始，是两个独立的 **翻译单元**。翻译单元由空行分隔，包含 **任意数量的注释和一个 msgid, msgstr对**。在翻译单元的注释中，一定会以 **文件名：行数** 的格式描述文本的引用位置。其他可能的信息还包括 **语言格式**，**翻译者注释** 等。

最后一个翻译单元如下：

```po
#: includes/admin/core/class-admin-upgrade.php:248
#, php-format
msgid "%s - Upgrade Process"
msgstr ""
```

第一行说明，待翻译的文本来自 `class-admin-upgrade.php` 源码文件的 248 行。后面的 `php-format` 注释表示字符串来自 **PHP** 源代码。

接下来是 msgid, msgstr对。跟在 msgid 后的是待翻译语句 `%s - Upgrade Process`，msgstr 后跟着的是空字符串，表示这个语句还没有被翻译。

msgid, msgstr对 还有多行的形式，例如：

```po
#: includes/admin/core/class-admin-notices.php:304
#, php-format
msgid ""
"%s needs to create several pages (User Profiles, Account, Registration, "
"Login, Password Reset, Logout, Member Directory) to function correctly."
msgstr ""
```

msgid 后分行跟随的几个字符串都是待翻译字串。理想情况下，在翻译完成后，msgstr 后应该跟随与 msgid 相同数量的翻译后句子，如下：

```po
#: includes/admin/core/class-admin-notices.php:304
#, php-format
msgid ""
"%s needs to create several pages (User Profiles, Account, Registration, "
"Login, Password Reset, Logout, Member Directory) to function correctly."
msgstr ""
"%s 需要创建一些页面（包括用户档案，账户，注册，"
"登录，密码重置，注销，成员字典）来保证正确运行。"
```

不管多大规模的 .po 文件，都是由这样形式的翻译单元组成。

另外，还有一种为支持语言单复数形式设计的 msgid 形式：

```po
#: includes/core/class-date-time.php:72
#, php-format
msgid "%s hr"
msgid_plural "%s hrs"
msgstr[0] ""
```



转换后的 .po 文件格式与 .pot 是一样的，唯一的不同是转换程序会根据转换的命令参数，填写 `Language` 等字段。一般翻译工作会在转换后的 .po 文件上进行，此自动翻译项目也不例外。