---
layout: post
title: "使用 Clang 解析 C++ 头文件实现自动脚本绑定"
date: 2015-01-03 15:30:06 +0800
comments: true
categoriesz: C/C++,clang,python,AST
---
最近为了方便工作，需要实现自动脚本绑定（所谓自动脚本绑定，即用 Lua、Python 之类的脚本语言来操作 C/C++ 的代码）的工具，因此首先要求无需手工编写任何绑定代码，此外还要考虑借助 Python 模板技术来批量生成代码，最终的目的是只需要在生成的框架上按需求添加业务逻辑。请注意，代码自动生成不是本博文的主旨，只是简单的演示脚本绑定例子。

对程序员这种一心只想偷懒的人来说，类似的重复工作都是写个程序来自动完成。下面介绍下我写的工具，它就是干这活儿的。
<!--more-->

工具的主要功能。它的输入是 C++ 头文件，里面包含了需要导出的 C++ 类定义。根据输入，会自动处理处理类/结构体的实现和一堆由模板文件生成的业务逻辑代码。

在工具内部，首先会对输入的 C++ 文件进行语法分析，生成抽象语法树(AST)。接着通过遍历语法树来生成所需要的包装导出代码。这里是使用访问者 (Visitor) 设计模式来解耦，把语法树的遍历逻辑和不同文件的代码生成逻辑区分开来，彼此独立实现。

这个工具究竟是怎么解析 C++ 文件的。众所周知，C++ 语法以繁复难以解析著称。此时有 3 个选择:

1. 手写解析器，对 C++ 语法有选择地分析。但这样做耗时耗力，而且很难避免出错;
2. 使用 pyparsing 等解析库，帮助我们实现简单的 C++ 语法解析器。这只比纯手写好上一点，难点还是在于 C++ 语法实在不是一般的复杂;
3. 利用真正的 C++ 编译器来解析。例如 g++ 就可以把解析后的语法树输出为 XML 结构，方便其它程序进一步利用。不少代码生成器就是这么做的;
比较权衡之后最终选择了第三种方式，使用一个真正的 C++ 编译器来帮助解析。但这里没有使用老牌的 g++，而是选择了另一位新晋明星: Clang。

> Clang 是苹果基于 LLVM 架构开发的 C++/Objective C 编译器，在被苹果 Xcode 加持后，又被 FreeBSD 选为默认编译器。势头正旺，大有取代 g++ 的架式。它不但对 C++ 新标准有完美的支持，更重要的是它把自己内部的语法解析功能通过 libclang 暴露出来，让用户能够直接使用。象 Xcode 集成开发环境的智能提示，还有一些第三方的 C++ 重构工具就是利用 Clang 本身提供的解析功能实现的。

本工具也正是利用 Clang 官方的 Python 扩展来实现 C++ 解析功能的。
一行程序胜过千言万语。下面的代码示范了如何利用 Clang 来解析输入的 C++ 头文件。

{% codeblock lang:python title:clang_3_local.py %}
#!/usr/bin/env python                                                                                                                                         
#coding=utf-8
import sys
import clang.cindex
from clang.cindex import Config
from clang.cindex import _CXString
Config.set_compatibility_check(False)
Config.set_library_path("/usr/lib")


def dumpnode(node, indent):
    '递归打印 C++ 语法树'
    # ...
    # do something with the current node here, i.e
    # check the kind, spelling, displayname and act baseed on those
    # kind     : 所查看的 AST 节点的类型, 该节点是类定义? 是函数定义? 还是参数声明？
    # spelling : 标记的文本定义. 比如游标指向 void foo(int x); 这样一个函数声明，该游标
    #            的 spelling 则是 foo.
    # ...
    text = node.spelling or node.displayname
    kind = str(node.kind)[str(node.kind).index('.')+1:]
    print ' ' * indent,'{} {}'.format(kind, text)
    for i in node.get_children():
        dumpnode(i, indent + 2)

def check_argv():
    if len(sys.argv) != 2:
        print("Usage: clang_3.py [header file name]")
        sys.exit()

def main():
    check_argv()
    index = clang.cindex.Index.create()
    tu = index.parse(sys.argv[1], ['-x', 'c++', '-std=c++11', '-D__CODE_GENERATOR__'])
    dumpnode(tu.cursor, 0)

if __name__ == '__main__':
    main()
{% endcodeblock %}

有了 Clang 的强力支持后，工具就可以不费吹灰之力就能解析任意 C++ 代码了。在用脚本操作类/结构体的场景中，上述代码十分典型。接下来就要考虑该如何使用解析出来的信息来自动生成所需要的代码。

查看头文件中的抽象语法树（AST），利用所获得的信息来生成绑定代码。脚本中是通过游标（cursor）来遍历 AST。游标指向 AST 中的某个节点，用来说明他是哪一类节点（比如说是类/结构体定义）其子节点是什么（比如说类/结构体的成员方法）等各类信息。第一个游标所指向的是 translation unit 的根(root)，即所要解析的文件。

以下的代码可以获得该游标：

{% codeblock lang:python title:cursor %}
index = clang.cindex.Index.create()
tu = index.parse(sys.argv[1], ['-x', 'c++', '-std=c++11', '-D__CODE_GENERATOR__'])
{% endcodeblock %}

index 对象是 libclang 的主要接口，由此就可以开始和类库进行通信了。

parse 函数的参数有文件名和包括各种编译标志的列表。这里需要制定要编译的是一个 C++ 头文件（-X C++），否则会根据 .h 的扩展名 libclang 会推断它是 C 的头文件（结果会产生大相径庭的 AST）。此选项会让 libclang 对指定的文件进行预处理（包括解析宏和 include），并将它作为 C++ 源码来处理。其他选项应该是不言自明的：设置源文件所遵循的 C++ 标准；并定义 __CODE_GENERATOR__宏，后续大部分实现代码都会用到的宏。最后传入的结构体/类就可以转储成对应的 AST。

下面是解析后的输出结果，大家可以直观感受一下 clang Python 扩展的威力:

{% codeblock lang:python title:output %}
   ...
   STRUCT_DECL SD_SignGlobal
     ANNOTATE_ATTR target
     FIELD_DECL viplevel
       ANNOTATE_ATTR get
       ANNOTATE_ATTR add
       TYPE_REF uint32_t
     FIELD_DECL propid
       TYPE_REF uint32_t
     FIELD_DECL num
       TYPE_REF uint32_t
     FIELD_DECL id
       ANNOTATE_ATTR hidden
       ANNOTATE_ATTR key
       TYPE_REF uint32_t
     FIELD_DECL month
       ANNOTATE_ATTR hidden
       ANNOTATE_ATTR key
       TYPE_REF uint32_t
     ...
{% endcodeblock %}

libclang C API 提供了基于访客（vistor-based）的方式来遍历 AST ，可以通过其 Python 绑定中的函数 clang.cindex.Cursor_vist 来遍历，不过更 Python 的方式是采用基于迭代器的方法，这是 Pythong绑定所特有的。处理树结构最直接的方式是递归遍历。这是相当简单的（见 dumpnode 函数）。

游标为 AST 中的每个节点提供了相当丰富的信息。通过游标的来分检出标记了的特定属性数据，然后就可以使用 Python 的模板工具生成响应的业务代码了。

现在工具还处于 alpha 阶段，虽然基本框架已经完成，但仍有很多改进的空间。除了继续完善工具的包装代码生成功能外，在现有架构下，拓展出更方便便捷的工具来。







