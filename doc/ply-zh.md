```C++
本文主要采用机翻译为主，但是通过了人工校对。 --- 20251023 

有一些关键词机翻不统一，这里进行统一说明翻译含义：
token  ： 符号
documentation string
```
<!-- TOC -->
* [PLY (Python Lex-Yacc)](#ply-python-lex-yacc)
  * [简介](#简介)
  * [PLY 概览](#ply-概览)
  * [Lex](#lex)
    * [Lex Example](#lex-example)
    * [符号列表](#符号列表)
    * [符号的规格](#符号的规格)
    * [符号的值](#符号的值)
    * [忽略符号](#忽略符号)
    * [行号和位置信息](#行号和位置信息)
    * [被忽略的字符](#被忽略的字符)
    * [字面字符](#字面字符)
    * [错误处理](#错误处理)
    * [换行处理](#换行处理)
    * [构建并使用词法分析器](#构建并使用词法分析器)
    * [\@TOKEN 装饰器](#token-装饰器)
    * [调试](#调试)
    * [词法解析器的可选规范](#词法解析器的可选规范)
    * [状态的维护](#状态的维护)
    * [克隆Lexer](#克隆lexer)
    * [词法解析器的内部状态](#词法解析器的内部状态)
    * [条件性解析和开始条件](#条件性解析和开始条件)
    * [其它](#其它)
  * [解析基础](#解析基础)
  * [Yacc](#yacc)
    * [示例](#示例)
    * [合并语法规则的功能](#合并语法规则的功能)
    * [Character Literals](#character-literals)
    * [空语句处理](#空语句处理)
    * [更改起始符号](#更改起始符号)
    * [应对模糊语法的情况](#应对模糊语法的情况)
    * [parser.out文件](#parserout文件)
    * [语法错误处理](#语法错误处理)
      * [使用错误规则进行回复和重新同步](#使用错误规则进行回复和重新同步)
      * [紧急模式恢复](#紧急模式恢复)
      * [从一个产品中报处错误信号](#从一个产品中报处错误信号)
      * [语法错误何时会被报告？](#语法错误何时会被报告)
      * [关于错误处理的一般性说明](#关于错误处理的一般性说明)
    * [行号和位置跟踪](#行号和位置跟踪)
    * [抽象语法树的构造](#抽象语法树的构造)
    * [嵌入式行为](#嵌入式行为)
    * [其它关于Yacc的注意事项](#其它关于yacc的注意事项)
  * [多个解析器和词法分析器](#多个解析器和词法分析器)
  * [高级调试](#高级调试)
    * [调试 lex() 和 yacc() 命令](#调试-lex-和-yacc-命令)
    * [运行时调试](#运行时调试)
  * [使用Python -OO 模式](#使用python--oo-模式)
  * [下一步如何做？](#下一步如何做)
<!-- TOC -->

# PLY (Python Lex-Yacc)

本文概述使用 PLY 进行词法分析和语法分析。
鉴于语法分析固有的复杂性，我强烈建议您在使用 PLY 开展大型开发项目之前通读（或至少浏览）整个文档。

当前版本需要 Python 3.6 或更高版本。如果您使用的是较旧版本的 Python，请使用其中一个历史版本。

## 简介

PLY 是一个纯 Python 实现的编译器构造工具 lex 和 yacc。
PLY 的主要目标是尽可能忠实于传统 lex/yacc 工具的工作方式。
这包括支持 LALR(1) 解析以及提供广泛的输入验证、错误报告和诊断功能。
因此，如果您在其他编程语言中使用过 yacc，那么使用 PLY 应该相对简单。

早期版本的 PLY 是为支持我在 2001 年于芝加哥大学教授的“编译器入门”课程而开发的。
由于PLY 主要被设计用作一种教学工具，您会发现它在符号和语法规则的设定方面相当严格。
在某种程度上，这种形式上的严谨性旨在发现新手用户在编程中常见的错误。
然而，高级用户也会发现这些功能在为真正的编程语言构建复杂语法规则时非常有用。
还应指出的是，PLY 并不提供太多花哨的功能（例如：自动构建抽象语法树、树遍历等）。
而且，我也不认为它是一个解析框架。相反，您会发现这是一个纯 Python 编写的、
功能简单但功能完备的 lex/yacc 实现。

本文其余部分假定您对解析理论、语法导向翻译以及在其他编程语言中使用诸如 lex 和
yacc 这类编译器构建工具有所了解。如果您对这些主题不熟悉，您可能会想要查阅诸如
《编译器：原理、技术与工具》（作者：Aho, Sethi, and Ullman）这样的入门书籍。
O'Reilly 出版的 John Levine所著的 《Lex 和 Yacc》也可能很有用。
实际上，O'Reilly 的这本书可以作为 PLY 的参考书，因为其中的概念几乎完全相同。

## PLY 概览

PLY 由两个独立的模块组成，分别是 `lex.py` 和 `yacc.py`，
它们都位于名为 `ply` 的 Python 包中。`lex.py` 模块用于将输入文本按照一系列
正则表达式规则所指定的方式分解为一系列符号。`yacc.py` 则用于识别以上下文无关语
法形式指定的语言语法。

这两款工具是相互协作使用的。具体而言，`lex.py` 提供了一个生成符号的接口。
`yacc.py` 利用此接口获取符号并调用语法规则。`yacc.py` 的输出通常是一个
抽象语法树（AST）。不过，这完全取决于用户的选择。如果需要的话，`yacc.py` 
还可以用于实现简单的单次编译器。

和 Unix 一样，`yacc.py` 也具备您所期望的大部分功能，包括全面的错误检查
、语法验证、支持空语句、错误标记以及通过优先级规则来解决歧义问题。实际上，传统 
yacc 中几乎所有可行的功能在 PLY 中都应该能够实现。

`yacc.py` 与 Unix 系统中的 `yacc` 之间的主要区别在于，`yacc.py` 不涉及
单独的代码生成过程。相反，PLY 利用反射（自我检查）来构建其词法分析器和解析器。
不像与传统的 lex/yacc 需要将输入文件转换为单独的源文件文件， PLY 给定的规范本身
就是有效的 Python 程序。这意味着没有额外的源文件，也没有特殊的编译器构建步骤
（例如，运行 yacc 以生成编译器的 Python 代码）。

## Lex

`lex.py` 用于对输入字符串进行词法分析。例如，假设您正在编写一种编程语言，
并且用户提供了以下输入字符串：

    x = 3 + 42 * (s - t)

词法分析器会将字符串拆分成一个个单独的符号：

    'x','=', '3', '+', '42', '*', '(', 's', '-', 't', ')'

符号会被赋予名称用以以表明其用途。例如：
    'ID','EQUALS','NUMBER','PLUS','NUMBER','TIMES',
    'LPAREN','ID','MINUS','ID','RPAREN'

更具体地说，输入内容会被拆分成一系列的符号的类型与值的组合。例如：

    ('ID','x'), ('EQUALS','='), ('NUMBER','3'), 
    ('PLUS','+'), ('NUMBER','42'), ('TIMES','*'),
    ('LPAREN','('), ('ID','s'), ('MINUS','-'),
    ('ID','t'), ('RPAREN',')')

符号的规格定义是通过编写一系列正则表达式规则来实现的。
下一节将展示如何使用 `lex.py` 来完成这一操作。

### Lex Example

以下示例展示了如何使用 `lex.py` 来编写一个简单的词法分析器：

    # ------------------------------------------------------------
    # calclex.py
    #
    # tokenizer for a simple expression evaluator for
    # numbers and +,-,*,/
    # ------------------------------------------------------------
    import ply.lex as lex

    # List of token names.   This is always required
    tokens = (
       'NUMBER',
       'PLUS',
       'MINUS',
       'TIMES',
       'DIVIDE',
       'LPAREN',
       'RPAREN',
    )

    # Regular expression rules for simple tokens
    t_PLUS    = r'\+'
    t_MINUS   = r'-'
    t_TIMES   = r'\*'
    t_DIVIDE  = r'/'
    t_LPAREN  = r'\('
    t_RPAREN  = r'\)'

    # A regular expression rule with some action code
    def t_NUMBER(t):
        r'\d+'
        t.value = int(t.value)    
        return t

    # Define a rule so we can track line numbers
    def t_newline(t):
        r'\n+'
        t.lexer.lineno += len(t.value)

    # A string containing ignored characters (spaces and tabs)
    t_ignore  = ' \t'

    # Error handling rule
    def t_error(t):
        print("Illegal character '%s'" % t.value[0])
        t.lexer.skip(1)

    # Build the lexer
    lexer = lex.lex()

要使用词法分析器，您首先需要通过其“input()”方法向其输入一些文本。
之后，多次调用“token()”方法即可生成符号。以下代码展示了其工作原理：

    # Test it out
    data = '''
    3 + 4 * 10
      + -20 *2
    '''

    # Give the lexer some input
    lexer.input(data)

    # Tokenize
    while True:
        tok = lexer.token()
        if not tok: 
            break      # No more input
        print(tok)

当执行该示例时，将会产生以下输出结果：

    $ python example.py
    LexToken(NUMBER,3,2,1)
    LexToken(PLUS,'+',2,3)
    LexToken(NUMBER,4,2,5)
    LexToken(TIMES,'*',2,7)
    LexToken(NUMBER,10,2,10)
    LexToken(PLUS,'+',3,14)
    LexToken(MINUS,'-',3,16)
    LexToken(NUMBER,20,3,18)
    LexToken(TIMES,'*',3,20)
    LexToken(NUMBER,2,3,21)

词法解析器还支持迭代协议。因此，您可以将上述循环写成如下形式：

    for tok in lexer:
        print(tok)

`lexer.token()` 方法返回的符号是 `LexToken` 类的实例。
该对象具有 `type`、`value`、`lineno` 和 `lexpos` 这些属性。
以下代码展示了访问这些属性的一个示例：

    # Tokenize
    while True:
        tok = lexer.token()
        if not tok: 
            break      # No more input
        print(tok.type, tok.value, tok.lineno, tok.lexpos)

“type”和“value”这两个属性分别包含了该符号的类型和值。
“lineno”和“lexpos”则包含了有关该符号位置的信息。
“lexpos”是该符号相对于输入文本起始位置的索引。

### 符号列表

所有解析器都必须提供一个名为“tokens”的列表，该列表定义了该解析器能够生成
的所有可能的符号名称。此列表始终是必需的，并用于执行各种验证检查。该标记列
表还被“yacc.py”模块用于识别符号。

在该示例中，以下代码指定了符号的名称：

    tokens = (
       'NUMBER',
       'PLUS',
       'MINUS',
       'TIMES',
       'DIVIDE',
       'LPAREN',
       'RPAREN',
    )

### 符号的规格
每个符号都是通过编写与 Python 的 `re` 模块兼容的正则表达式来指定的。
这些规则都是通过带有特殊前缀“t_”的声明来定义的，该前缀用于表明它定义了一个符号。
对于简单的符号，正则表达式可以以这样的字符串形式指定（注意：由于使用的是 Python 的
原始字符串，所以这是编写正则表达式字符串最方便的方式）：

    t_PLUS = r'\+'

在这种情况下，以“t_”开头的名称必须与“tokens”中提供的名称完全一致。如果需要执行某种操作，
可以将一个令牌规则指定为一个函数。例如，此规则将匹配数字，并将字符串转换为 Python 整数：

    def t_NUMBER(t):
        r'\d+'
        t.value = int(t.value)
        return t

当使用一个函数时，正则表达式规则会包含在该函数中。该函数总是接受一个参数，
该参数是一个 `LexToken` 类型的对象。此对象具有以下属性：`type`，即符号类型（以字符
串形式表示）；`value`，即词法单元（匹配的实际文本）；`lineno`，即当前行号；以及
`lexpos`，即标记相对于输入文本起始位置的偏移量。默认情况下，`type` 被设置为带有
`t_` 前缀的名称。行为函数可以根据需要修改 `LexToken` 对象的内容。然而，在完成修
改后，应返回生成的标记。如果动作函数未返回任何值，则该符号将被丢弃，并读取下一个标记。

在内部，`lex.py` 使用 `re` 模块来进行模式匹配。模式是通过使用 `re.VERBOSE` 标志
进行编译的，该标志有助于提高可读性。但请注意，未转义的空格会被忽略，而且在此模式下允许
添加注释。如果您的模式包含空格，请确保使用 `\s` 。如果您需要匹配 `#` 字符，请使用
`[#]` 。

在构建主（译：？这里翻译不太对）正则表达式时，规则的添加顺序如下：

1.  由函数定义的所有符号都将按照它们在词法分析器文件中的出现顺序进行添加。
2.  对由字符串定义的符号会将排序后的内容添加进来（即先添加长度较长的表达式）。

如果没有这种排序方式，就可能难以正确匹配某些特定类型的符号。例如，如果您想要为“=”和“==”分别
设置不同的标记，那么就需要确保先检查“==”。通过按照长度递减的顺序对正则表达式进行排序，对
于定义为字符串的规则，这个问题就能得到解决。对于函数，由于最先出现的规则会被优先检查，所
以其顺序可以明确地加以控制。

要处理保留字，您需要编写一条规则来匹配标识符，并在函数中进行特殊的名称查找操作，例如：

To handle reserved words, you should write a single rule to match an
identifier and do a special name lookup in a function like this:

    reserved = {
       'if' : 'IF',
       'then' : 'THEN',
       'else' : 'ELSE',
       'while' : 'WHILE',
       ...
    }

    tokens = ['LPAREN','RPAREN',...,'ID'] + list(reserved.values())

    def t_ID(t):
        r'[a-zA-Z_][a-zA-Z_0-9]*'
        t.type = reserved.get(t.value,'ID')    # Check for reserved words
        return t

这种方法大大减少了正则表达式规则的数量，并且可能会使操作速度有所提升。

注意：您应避免为保留词汇单独制定规则。例如，如果你这样制定规则：

    t_FOR   = r'for'
    t_PRINT = r'print'

这些规则会在包含上述词汇作为前缀的标识符（如“忘记”或“打印”）的情况下被触发。
这可能并非您所期望的结果。

### 符号的值

当 lex 函数返回符号时，这些符号会有一个值，该值存储在“value”属性中。通常，该值就是匹配到
的文本。然而，该值可以赋给任何 Python 对象。例如，在对标识符进行 lex 时，您可能希望返回
标识符名称以及某种符号表中的相关信息。为此，您可以编写这样的规则：

    def t_ID(t):
        ...
        # Look up symbol table information and return a tuple
        t.value = (t.value, symbol_lookup(t.value))
        ...
        return t

需要特别注意的是，在其他属性名称中存储数据是不被推荐的。`yacc.py` 模块仅暴露 `value` 
属性的内容。因此，访问其他属性可能会显得非常不方便。如果需要在一个标记上存储多个值，可以
将一个元组、字典或实例赋值给 `value` 。

### 忽略符号

若要忽略某个符号（例如注释），请定义一个不返回任何值的标记规则。例如：

    def t_COMMENT(t):
        r'\#.*'
        pass
        # No return value. Token discarded

或者，您可以在符号声明中添加前缀“ignore_”，以强制忽略该符号。例如：

    t_ignore_COMMENT = r'\#.*'

请注意，如果你忽略不同种类的文本，您可能仍需要使用函数，因为这些函数能提供更精确的控制，
以确保正则表达式匹配的顺序（即函数是按照定义的顺序进行匹配的，而字符串则是按照正则表达式的
长度进行排序的）。

### 行号和位置信息

默认情况下，`lex.py` 不知道行号。这是因为 `lex.py` 并不知道输入数据中“一行”的构成
要素（例如，换行符，甚至输入是否为文本数据）。要更新这些信息，您需要编写一个特殊的规则。
在示例中，`t_newline()` 规则展示了如何进行此操作：

    # Define a rule so we can track line numbers
    def t_newline(t):
        r'\n+'
        t.lexer.lineno += len(t.value)

在该规则中，底层词法解析器 `t.lexer` 的 `lineno` 属性会被更新。在行号更新完成后，
该符号将被丢弃，因为不会再返回任何内容。

`lex.py` 并不执行任何自动列跟踪的功能。不过，它会在 `lexpos` 属性中记录每个标记的
相关位置信息。利用这些信息，通常可以将列信息的计算作为单独的步骤来进行。例如，只需从当
前位置向后数直到遇到换行符即可：

    # Compute column.
    #     input is the input text string
    #     token is a token instance
    def find_column(input, token):
        line_start = input.rfind('\n', 0, token.lexpos) + 1
        return (token.lexpos - line_start) + 1

由于列信息通常仅在错误处理的上下文中才有用，因此可以按需计算列位置，而不是为每个标记都进
行计算。注意：如果你正在解析一种对空格有特定要求的语言（例如 Python），那么最好将空格作
为标记进行匹配，而不是忽略它。

### 被忽略的字符

特殊的`t_ignore`规则是由`lex.py`模块专门用于标识在输入流中应该被完全忽略的字符的。
通常，这一规则用于跳过空格和其他非必要的字符。尽管可以采用类似于`t_newline()`的方式
为空格定义一个正则表达式规则，但使用`t_ignore`能提供显著更高的词法解析性能，因为它被
当作一个特殊情况处理，并且其检查方式比常规的正则表达式规则要高效得多。

在 `t_ignore` 中指定的字符在这些字符属于其他正则表达式模式的一部分时不会被忽略。
例如，如果有一个规则用于捕获带引号的文本，那么该模式可以包含被忽略的字符（这些字符
将以常规方式被捕获）。`t_ignore` 的主要用途是忽略您实际想要解析的符号之间的空白
字符和其他填充字符。

### 字面字符

可以通过在你的词法解析模块中定义一个名为`literals`的变量来指定字面字符。例如：

    literals = [ '+','-','*','/' ]

或者:

    literals = "+-*/"

`字面字符`是指在解析器遇到时会\"as is\"返回的单个字符。在所有已定义的正则表达式规则检查
完毕后，会对这些字面字符进行验证。因此，如果某个规则以这些字面字符中的任何一个开头，那
么它将始终具有优先级。

当返回一个字面字符符号时，其`type`和`value`属性都会被设置为该字符本身。例如，`'+'`。

可以编写一些符号函数，以便在匹配到特定的字面值时执行额外的操作。
不过，您需要正确设置标记类型。例如：

    literals = [ '{', '}' ]

    def t_lbrace(t):
        r'\{'
        t.type = '{'      # Set token type to the expected literal
        return t

    def t_rbrace(t):
        r'\}'
        t.type = '}'      # Set token type to the expected literal
        return t

### 错误处理

`t_error()` 函数用于处理在检测到非法字符时出现的词法错误。在这种情况下，`t.value` 
属性包含了尚未被分词处理的输入字符串的剩余部分。在示例中，错误处理函数的定义如下：

    # Error handling rule
    def t_error(t):
        print("Illegal character '%s'" % t.value[0])
        t.lexer.skip(1)

在这种情况下，我们会打印出错误字符，并通过调用 `t.lexer.skip(1)` 机制跳过下一个字符。

### 换行处理

`t_eof()` 函数被用于处理输入中的文件结束（EOF）情况。在输入中，它接收一个符号类型为 `'eof'` 
的参数，并将 `lineno` 和 `lexpos` 属性设置为适当的值。此函数的主要用途是为词法分析器提供
更多输入，以便其能够继续解析。以下是其工作方式的一个示例：

    # EOF handling rule
    def t_eof(t):
        # Get more input (Example)
        more = input('... ')
        if more:
            t.lexer.input(more)
            return t.lexer.token()
        return None

EOF 函数应返回下一个可用的符号（通过调用 `t.lexer.token()` 来实现）或者返回 `None` 
以表示没有更多数据。请注意，使用 `t.lexer.input()` 方法添加更多输入并不会重置解析器状
态或用于位置跟踪的 `lineno` 属性。`lexpos` 属性会被重置，所以在错误报告中使用该属性时
请务必注意这一点。

### 构建并使用词法分析器

要构建词法分析器，需要使用函数 `lex.lex()` 。例如：

    lexer = lex.lex()

此功能利用 Python 的反射（或内检）功能从调用环境中读取正则表达式规则，并构建词法分析器。
一旦词法分析器构建完成，就可以使用两种方法来控制该词法分析器：

`lexer.input(data)`。重置解析器并存储新的输入字符串。

`lexer.token()` 返回下一个符号。成功时会返回一个特殊的 `LexToken` 实例，
若已到达输入文本的末尾则返回 None 。

### \@TOKEN 装饰器

在某些应用场景中，您可能希望将令牌定义为一系列更为复杂的正则表达式规则。例如：

    digit            = r'([0-9])'
    nondigit         = r'([_A-Za-z])'
    identifier       = r'(' + nondigit + r'(' + digit + r'|' + nondigit + r')*)'        

    def t_ID(t):
        # want docstring to be identifier above. ?????
        ...

在这种情况下，我们希望 `ID` 的正则表达式规则属于上述变量之一。然而，无法通过常规的文档字
符串直接指定这一点。为了解决这个问题，您可以使用 `@TOKEN` 装饰器。例如：

    from ply.lex import TOKEN

    @TOKEN(identifier)
    def t_ID(t):
        ...

这会将`identifier`附加到 `t_ID()` 的文档字符串中，从而使 `lex.py` 能够正常运行。当然，
您也可以对所有函数使用 `@TOKEN` 作为使用文档字符串的替代方法。(译：这个可能需要试一下)

### 调试

为了进行调试，您可以以调试模式运行 `lex()` 函数，具体操作如下：

    lexer = lex.lex(debug=True)

这将生成各种类型的调试信息，包括所有添加的规则、词法分析器所使用的主正则表达式以及在词法分
析过程中生成的标记。

此外，`lex.py` 附带了一个简单的主函数，该函数能够对从标准输入读取的输入内容或从命令行指
定的文件中读取的内容进行分词处理。要使用它，请将以下代码段添加到你的分词器中：

    if __name__ == '__main__':
         lex.runmain()

请参阅文末的\"调试\"部分，以获取更多有关调试的高级细节说明。

### 词法解析器的可选规范

如示例所示，词法分析器全部都在一个 Python 模块中进行定义。如果您想将标记规则放在与调
用 `lex()` 的模块不同的模块中，请使用 `module` 关键字参数。

例如，你可能会有一个专门的模块，其中仅包含符号规则：

    # module: tokrules.py
    # This module just contains the lexing rules

    # List of token names.   This is always required
    tokens = (
       'NUMBER',
       'PLUS',
       'MINUS',
       'TIMES',
       'DIVIDE',
       'LPAREN',
       'RPAREN',
    )

    # Regular expression rules for simple tokens
    t_PLUS    = r'\+'
    t_MINUS   = r'-'
    t_TIMES   = r'\*'
    t_DIVIDE  = r'/'
    t_LPAREN  = r'\('
    t_RPAREN  = r'\)'

    # A regular expression rule with some action code
    def t_NUMBER(t):
        r'\d+'
        t.value = int(t.value)    
        return t

    # Define a rule so we can track line numbers
    def t_newline(t):
        r'\n+'
        t.lexer.lineno += len(t.value)

    # A string containing ignored characters (spaces and tabs)
    t_ignore  = ' \t'

    # Error handling rule
    def t_error(t):
        print("Illegal character '%s'" % t.value[0])
        t.lexer.skip(1)

现在，如果您想在另一个模块中根据这些规则构建一个分词器，您可以按照以下步骤操作
（此处以 Python 交互模式为例）：

    >>> import tokrules
    >>> lexer = lex.lex(module=tokrules)
    >>> lexer.input("3 + 4")
    >>> lexer.token()
    LexToken(NUMBER,3,1,1,0)
    >>> lexer.token()
    LexToken(PLUS,'+',1,2)
    >>> lexer.token()
    LexToken(NUMBER,4,1,4)
    >>> lexer.token()
    None
    >>>

`module` 选项还可用于通过类的实例来定义解析器。例如：

    import ply.lex as lex

    class MyLexer(object):
        # List of token names.   This is always required
        tokens = (
           'NUMBER',
           'PLUS',
           'MINUS',
           'TIMES',
           'DIVIDE',
           'LPAREN',
           'RPAREN',
        )

        # Regular expression rules for simple tokens
        t_PLUS    = r'\+'
        t_MINUS   = r'-'
        t_TIMES   = r'\*'
        t_DIVIDE  = r'/'
        t_LPAREN  = r'\('
        t_RPAREN  = r'\)'

        # A regular expression rule with some action code
        # Note addition of self parameter since we're in a class
        def t_NUMBER(self,t):
            r'\d+'
            t.value = int(t.value)    
            return t

        # Define a rule so we can track line numbers
        def t_newline(self,t):
            r'\n+'
            t.lexer.lineno += len(t.value)

        # A string containing ignored characters (spaces and tabs)
        t_ignore  = ' \t'

        # Error handling rule
        def t_error(self,t):
            print("Illegal character '%s'" % t.value[0])
            t.lexer.skip(1)

        # Build the lexer
        def build(self,**kwargs):
            self.lexer = lex.lex(module=self, **kwargs)

        # Test it output
        def test(self,data):
            self.lexer.input(data)
            while True:
                 tok = self.lexer.token()
                 if not tok: 
                     break
                 print(tok)

    # Build the lexer and try it out
    m = MyLexer()
    m.build()           # Build the lexer
    m.test("3 + 4")     # Test it

在使用类来构建词法分析器时，*您应当从该类的实例而非类对象本身来构建词法分析器*。这是因为
PLY 只有在词法操作由绑定方法定义的情况下才能正常工作。

在使用 `lex()` 函数的 `module` 选项时，PLY 会通过 `dir()` 函数从底层对象中收集符号。
但无法直接访问作为模块值提供的对象的 `__dict__` 属性。

最后，如果您希望将内容妥善封装起来，但又不想使用完整的类定义，那么可以使用闭包来定义解析器。例如：

    import ply.lex as lex

    # List of token names.   This is always required
    tokens = (
      'NUMBER',
      'PLUS',
      'MINUS',
      'TIMES',
      'DIVIDE',
      'LPAREN',
      'RPAREN',
    )

    def MyLexer():
        # Regular expression rules for simple tokens
        t_PLUS    = r'\+'
        t_MINUS   = r'-'
        t_TIMES   = r'\*'
        t_DIVIDE  = r'/'
        t_LPAREN  = r'\('
        t_RPAREN  = r'\)'

        # A regular expression rule with some action code
        def t_NUMBER(t):
            r'\d+'
            t.value = int(t.value)    
            return t

        # Define a rule so we can track line numbers
        def t_newline(t):
            r'\n+'
            t.lexer.lineno += len(t.value)

        # A string containing ignored characters (spaces and tabs)
        t_ignore  = ' \t'

        # Error handling rule
        def t_error(t):
            print("Illegal character '%s'" % t.value[0])
            t.lexer.skip(1)

        # Build the lexer from my environment and return it    
        return lex.lex()

重要提示：如果您使用类或闭包来定义词法分析器，请注意，PLY 仍然要求您每个模块
（源文件）仅定义一个词法分析器。PLY 中有大量验证/错误检查部分，如果不遵循此
规则，可能会错误地报告错误消息。

### 状态的维护

在你的词法分析器中，你可能需要保存多种状态信息。这可能包括模式设置、符号表以及其他细节。
例如，假设你想要记录已经遇到的 NUMBER 标记的数量。

一种实现此目的的方法是在创建词法分析器的模块中定义一组全局变量。例如：

    num_count = 0
    def t_NUMBER(t):
        r'\d+'
        global num_count
        num_count += 1
        t.value = int(t.value)    
        return t

如果您不喜欢使用全局变量，那么还可以将信息存储在由 `lex()` 函数创建的词法分析器的对象内部。
要做到这一点，您可以使用传递给各种规则的符号的 `lex()` 属性。例如：

    def t_NUMBER(t):
        r'\d+'
        t.lexer.num_count += 1     # Note the use of lexer attribute
        t.value = int(t.value)    
        return t

    lexer = lex.lex()
    lexer.num_count = 0            # Set the initial count

后一种方法的优点在于其简单易行，并且在同一个应用程序中存在给定解析器的多个实例的情况下也能正常
工作。然而，对于面向对象的纯粹主义者来说，这可能看起来是对封装的严重违背。为了让您安心，解析器
的所有内部属性（除了 `lineno` 之外）都以 `lex` 为前缀命名（例如，`lexdata`、`lexpos` 
等）。因此，将那些名称不以该前缀开头或与预定义方法（如 `input()`、`token()` 等）冲突的属性
存储在解析器中是完全安全的。

如果您不想在词法解析器对象上赋值，您可以像上一节中所示那样将词法解析器定义为一个类：

    class MyLexer:
        ...
        def t_NUMBER(self,t):
            r'\d+'
            self.num_count += 1
            t.value = int(t.value)    
            return t

        def build(self, **kwargs):
            self.lexer = lex.lex(object=self,**kwargs)

        def __init__(self):
            self.num_count = 0

如果您的应用程序需要创建多个相同类型的词法分析器实例，并且需要管理
大量状态的话，那么采用类方法可能会是最容易管理的方式。

状态也可以通过闭包来进行管理。例如：

    def MyLexer():
        num_count = 0
        ...
        def t_NUMBER(t):
            r'\d+'
            nonlocal num_count
            num_count += 1
            t.value = int(t.value)    
            return t
        ...

### 克隆Lexer

If necessary, a lexer object can be duplicated by invoking its `clone()`
method. For example:
如果需要，可以通过调用其`clone()`方法来复制一个词法分析器对象。例如：

    lexer = lex.lex()
    ...
    newlexer = lexer.clone()

当一个词法分析器被克隆时，其副本与原始词法分析器完全相同，包括任何输入文本和内部状态。
然而，该克隆允许提供不同的输入文本集，这些文本可以单独进行处理。这在您编写涉及递归
或可重入处理的解析器/编译器时可能非常有用。例如，如果出于某种原因需要提前扫描输入内容，
您可以创建一个克隆并使用它来进行提前扫描。或者，如果您正在实现某种重处理器，可以使用
克隆来处理不同的输入文件。

创建克隆与调用 `lex.lex()` 有所不同，因为 PLY 不会重新生成任何内部表或正则表达式。

在对同时使用类或闭包来维护自身内部状态的解析器进行克隆时，需要特别注意一些事项。具体来说，
您需要明白新创建的解析器将与原始解析器共享所有这些状态。例如，如果您将解析器定义为一个类，
并执行以下操作：

    m = MyLexer()
    a = lex.lex(object=m)      # Create a lexer

    b = a.clone()              # Clone the lexer

然后，变量 `a` 和 `b` 都将被绑定到同一个对象 `m` 上，对 `m` 的任何更改都会反映在两个解
析器中。需要强调的是，`clone()` 仅用于创建一个新的解析器，该解析器会复用另一个解析器的正
则表达式和环境。如果您需要完全复制一个解析器，则需要再次调用 `lex()` 。

### 词法解析器的内部状态

一个名为`lexer`的词法分析器对象具有若干内部属性，在某些情况下这些属性可能会派上用场：

`lexer.lexpos`

:   此属性为一个整数，用于表示输入文本中的当前位置。若修改其值，则会改变对 `token()` 
    函数的下一次调用的结果。在符号规则函数中，此指针指向匹配文本之后的第一个字符。若在规
    则内部修改该值，则下一次返回的符号将在新的位置进行匹配。

`lexer.lineno`

:   当前存储在词法分析器中的行号属性的值。PLY 仅规定该属性存在——它从不设置、更新或对其进行
    任何处理。如果您想要跟踪行号，您需要自己添加代码（请参阅关于行号和位置信息的章节）。

`lexer.lexdata`

:   当前存储在词法分析器中的输入文本。这是通过 `input()` 方法传递的字符串。除非您非常清
    楚自己在做什么，否则最好不要对其进行修改。

`lexer.lexmatch`

:   这是由 Python 的 `re.match()` 函数（PLY 内部使用的函数）为当前标记返回的原始`Match`
    对象。如果您编写了包含命名组的正则表达式，就可以使用此对象来获取这些值。
    注意：此属性仅在通过函数定义和处理符号时才会更新。

### 条件性解析和开始条件

在高级解析应用中，设置不同的词法状态可能会很有用。例如，您可能希望某个特定的标记或语法结构
的出现能够触发一种不同的词法处理方式。PLY 支持一种功能，允许底层的词法解析器进入一系列不同
的状态。每个状态都可以有自己的标记、词法规则等等。其实现主要基于 GNU flex 的\"start
condition\"特性。相关详情请见以下内容：
<https://westes.github.io/flex/manual/Start-Conditions.html>

要定义一个新的词法状态，首先必须对其进行声明。这可以通过在词法文件中添加\"states\"声明来实现。
例如：

    states = (
       ('foo','exclusive'),
       ('bar','inclusive'),
    )

此声明定义了两个状态，分别为`'foo'`和`'bar'`。状态可分为两种类型：`'exclusive'`和`'inclusive'`。
一个``'exclusive'``状态会完全取代词法解析器的默认行为。也就是说，lex 只会返回特定于该状态的标记，
并应用针对该状态定义的规则。一个``'inclusive'``状态会向默认规则集添加额外的标记和规则。
因此，lex 除了返回默认定义的标记外，还会返回针对“包含型”状态定义的标记。

一旦声明了state，就可以通过在符号/规则声明中包含状态名称来定义令牌和规则。例如：

    t_foo_NUMBER = r'\d+'                      # Token 'NUMBER' in state 'foo'        
    t_bar_ID     = r'[a-zA-Z_][a-zA-Z0-9_]*'   # Token 'ID' in state 'bar'

    def t_foo_newline(t):
        r'\n'
        t.lexer.lineno += 1

可以通过在声明中包含多个状态名称来为一个符号定义多个状态。例如：

    t_foo_bar_NUMBER = r'\d+'         # Defines token 'NUMBER' in both state 'foo' and 'bar'

另外，可以通过在名称中使用 \'ANY\'关键字在所有状态中声明一个标记：

    t_ANY_NUMBER = r'\d+'         # Defines a token 'NUMBER' in all states

如果未提供任何状态名（这是通常的情况），那么该标记将与一个特殊的状态`'INITIAL'`相关联。
例如，以下这两段声明是完全相同的：

    t_NUMBER = r'\d+'
    t_INITIAL_NUMBER = r'\d+'

此外，状态还与特殊的`t_ignore`、`t_error()`和`t_eof()`声明相关联。例如，如果某个状态
对这些情况有不同的处理方式，您可以这样声明：

    t_foo_ignore = " \t\n"       # Ignored characters for state 'foo'

    def t_bar_error(t):          # Special error handler for state 'bar'
        pass 

默认情况下，词法分析处于`'INITIAL'`状态。此状态包含了所有通常定义的标记。对于未使用不同状态的用户
而言，这一事实完全不会引起注意。如果在词法分析或解析过程中您想要更改词法分析状态，请使用 `begin()` 方法。例如：

    def t_begin_foo(t):
        r'start_foo'
        t.lexer.begin('foo')             # Starts 'foo' state

要退出当前状态，可以使用 `begin()` 方法切换回初始状态。例如：

    def t_foo_end(t):
        r'end_foo'
        t.lexer.begin('INITIAL')        # Back to the initial state

状态的管理也可以通过栈来实现。例如：

    def t_begin_foo(t):
        r'start_foo'
        t.lexer.push_state('foo')             # Starts 'foo' state

    def t_foo_end(t):
        r'end_foo'
        t.lexer.pop_state()                   # Back to the previous state

在存在多种方式可以进入新的词法状态，并且您只是希望之后能返回到之前状态的情况下，使用栈会非常有用。

举个例子或许能更清楚地说明问题。假设你正在编写一个解析器，并且想要提取由花括号括起来的任意 C 
代码段。也就是说，每当遇到一个起始花括号``{``时，你就希望读取其内部的所有代码直到结束花括号``}``，
并将其作为字符串返回。使用普通的正则表达式规则来实现这一点几乎是不可能的（即便实际上也是不可能的）。
这是因为花括号可以嵌套，并且可以包含在注释和字符串中。因此，仅仅匹配到第一个匹配的``}``字符是不够的。
以下是您可以如何使用词法状态来实现这一点的方法：

	import ply.lex as lex

	# Declare the states
    states = (
      ('ccode','exclusive'),
    )

    # Match the first '{' Enter ccode state.
    def t_ccode(t):
        r'\{'
        t.lexer.code_start = t.lexer.lexpos        # Record the starting position
        t.lexer.level = 1                          # Initial brace level
        t.lexer.begin('ccode')                     # Enter 'ccode' state

    # Rules for the 'ccode' state
    def t_ccode_lbrace(t):     
        r'\{'
        t.lexer.level += 1                

    def t_ccode_rbrace(t):
        r'\}'
        t.lexer.level -= 1

        # If closing brace, return the code fragment
        if t.lexer.level == 0:
             t.value = t.lexer.lexdata[t.lexer.code_start:t.lexer.lexpos+1]
             t.type = "CCODE"
             t.lexer.lineno += t.value.count('\n')
             t.lexer.begin('INITIAL')           
             return t

    # C or C++ comment (ignore)    
    def t_ccode_comment(t):
        r'(/\*(.|\n)*?\*/)|(//.*)'
        pass

    # C string
    def t_ccode_string(t):
       r'\"([^\\\n]|(\\.))*?\"'

    # C character literal
    def t_ccode_char(t):
       r'\'([^\\\n]|(\\.))*?\''

    # Any sequence of non-whitespace characters (not braces, strings)
    def t_ccode_nonspace(t):
       r'[^\s\{\}\'\"]+'

    # Ignored characters (whitespace)
    t_ccode_ignore = " \t\n"

    # For bad characters, we just skip over it
    def t_ccode_error(t):
        t.lexer.skip(1)
	
	lexer = lex.lex()
    data = "{}"

    lexer.input(data)
    while True:
        tok = lexer.token()
        if not tok:
            break
        print(tok)


在该示例中，第一个``{``符号的出现致使词法分析器记录下起始位置，并进入新的状态`'ccode'`。
随后会有一系列规则来匹配输入中后续的各种部分（注释、字符串等）。这些规则只是丢弃该标记
（不返回任何值）。然而，如果遇到右大括号，则规则 `t_ccode_rbrace` 会收集所有代码
（使用之前记录的起始位置），将其存储起来，并返回一个标记 'CCODE'，其中包含所有这些文本。
在返回该标记时，解析状态会恢复到初始状态。

### 其它
 
-   词法分析器要求输入以单个输入字符串的形式提供。由于大多数机器的内存都绰绰有余，因此这通
    常不会造成性能问题。然而，这意味着词法分析器目前无法用于诸如打开的文件或套接字之类的流数据。
    这一限制主要是由于使用 `re` 模块所导致的。您或许可以通过实现适当的 `def t_eof()` 结束
    文件处理规则来解决这个问题。这里的主要复杂之处在于，您可能需要确保将数据以某种方式提供给词法
    分析器，以避免在标记中间进行分割。

-   如果您需要向 ``re.compile()`` 函数提供可选的标志，请向 lex 传递 ``reflags`` 选项。
    例如：

        lex.lex(reflags=re.UNICODE | re.VERBOSE)

    注意：默认情况下，`reflags` 被设置为 `re.VERBOSE`。如果您自定义了标志，则可能需要
    将此设置包含在内，以便 PLY 能够保持其正常行为。

-   如果你打算创建一个手写词法分析器，并且计划将其与 `yacc.py` 结合使用，那么它只需要满
    足以下这些要求：

    1.  必须提供一个 `token()` 方法，该方法能够返回下一个标记，如果没有更多标记则返回 `None` 。
    2.  `token()` 方法必须返回一个具有 `type` 和 `value` 属性的对象 `tok` 。 
        如果正在使用行号跟踪功能，那么该标记还应定义一个 `lineno` 属性。

## 解析基础

`yacc.py` 用于解析语言的语法结构。在展示示例之前，有必要先提及一些重要的背景信息。
首先，*语法*通常是以 BNF 语法的形式来定义的。例如，如果您想要解析简单的算术表达式，您
可能会首先编写一个清晰明确的语法规范，如下所示：

    expression : expression + term
               | expression - term
               | term

    term       : term * factor
               | term / factor
               | factor

    factor     : NUMBER
               | ( expression )

在语法中，诸如 `NUMBER`, `+`, `-`, `*`, 和 `/` 这样的符号被称为*terminals*，
它们对应于输入的标记。诸如`term`和`factor`这样的标识符则代表由一系列终结符及其他规则组成的语法规则。
这些标识符被称为*non-terminals*。

一种语言的语义行为通常通过一种称为语法导向翻译的技术来加以规定。在语法导向翻译中，
会在给定的语法规则中的每个符号上附加属性以及相应的操作。每当识别到特定的语法规则时，
该操作就会说明要执行的操作。例如，对于上述表达式语法，您可以这样为一个简单的计算器编写规范：

    Grammar                             Action
    --------------------------------    -------------------------------------------- 
    expression0 : expression1 + term    expression0.val = expression1.val + term.val
                | expression1 - term    expression0.val = expression1.val - term.val
                | term                  expression0.val = term.val

    term0       : term1 * factor        term0.val = term1.val * factor.val
                | term1 / factor        term0.val = term1.val / factor.val
                | factor                term0.val = factor.val

    factor      : NUMBER                factor.val = int(NUMBER.lexval)
                | ( expression )        factor.val = expression.val

一种思考语法导向翻译的好方法是将语法中的每个符号视为一种对象。与每个符号相关联的是一个表
示其\"state\"的值（例如，上面的 `val` 属性）。语义动作则表示为一组作用于符号及其相关
值的函数或方法。

Yacc 采用一种被称为 LR 分析或移位-归约分析的解析技术。LR 分析是一种自下而上的技术，旨
在识别各种语法规则的右侧内容。每当在输入中找到有效的右侧内容时，就会触发相应的操作代码，
并将语法符号替换为左侧的语法符号。

LR 分析通常通过将语法符号压入栈中，并查看栈以及下一个输入标记来寻找与语法规则匹配的模式来
实现。该算法的详细内容可在编译器教材中找到，但以下示例说明了如果要使用上述定义的语法来解析
表达式“3 + 5 * (10 - 20)”时所执行的步骤。在该示例中，特殊符号“$”表示输入的结束标志：

    Step Symbol Stack           Input Tokens            Action
    ---- ---------------------  ---------------------   -------------------------------
    1                           3 + 5 * ( 10 - 20 )$    Shift 3
    2    3                        + 5 * ( 10 - 20 )$    Reduce factor : NUMBER
    3    factor                   + 5 * ( 10 - 20 )$    Reduce term   : factor
    4    term                     + 5 * ( 10 - 20 )$    Reduce expr : term
    5    expr                     + 5 * ( 10 - 20 )$    Shift +
    6    expr +                     5 * ( 10 - 20 )$    Shift 5
    7    expr + 5                     * ( 10 - 20 )$    Reduce factor : NUMBER
    8    expr + factor                * ( 10 - 20 )$    Reduce term   : factor
    9    expr + term                  * ( 10 - 20 )$    Shift *
    10   expr + term *                  ( 10 - 20 )$    Shift (
    11   expr + term * (                  10 - 20 )$    Shift 10
    12   expr + term * ( 10                  - 20 )$    Reduce factor : NUMBER
    13   expr + term * ( factor              - 20 )$    Reduce term : factor
    14   expr + term * ( term                - 20 )$    Reduce expr : term
    15   expr + term * ( expr                - 20 )$    Shift -
    16   expr + term * ( expr -                20 )$    Shift 20
    17   expr + term * ( expr - 20                )$    Reduce factor : NUMBER
    18   expr + term * ( expr - factor            )$    Reduce term : factor
    19   expr + term * ( expr - term              )$    Reduce expr : expr - term
    20   expr + term * ( expr                     )$    Shift )
    21   expr + term * ( expr )                    $    Reduce factor : (expr)
    22   expr + term * factor                      $    Reduce term : term * factor
    23   expr + term                               $    Reduce expr : expr + term
    24   expr                                      $    Reduce expr
    25                                             $    Success!

在解析表达式时，底层的状态机和当前输入的标记将决定接下来会发生什么。如果下一个标记看起来像
是一个有效的语法规则的一部分（基于栈上的其他项目），通常会将其移到栈中。如果栈顶包含一个语
法规则的有效右侧部分，通常会对其进行“简化”，并将左侧的符号替换为右侧的符号。当这种简化发生
时，会触发相应的操作（如果已定义）。如果输入标记无法移除且栈顶与任何语法规则都不匹配，就会
发生语法错误，解析器必须采取某种恢复步骤（或终止）。只有当解析器到达符号栈为空且没有更多输
入标记的状态时，解析才是成功的。

需要特别注意的是，其底层实现是基于一个大型的有限状态机构建的，该状态机被编码在一系列表格
中。这些表格的构建过程并不简单，超出了本次讨论的范围。然而，这一过程中的细微之处能够解释
为何在上述示例中，解析器在第 9 步选择将一个标记推到栈中，而不是对规则“expr ： expr + term”
进行归约。

## Yacc

`ply.yacc` 模块实现了 PLY 的解析组件。该模块的名称“yacc”源自“Yet Another Compiler Compiler”
（又一个编译器编译器），这一名称借鉴自 Unix 系统中的同名工具。

### 示例

假设您想要按照之前所述的方式为简单的算术表达式创建一个语法规则。下面是使用 `yacc.py` 实现的方法：

    # Yacc example

    import ply.yacc as yacc

    # Get the token map from the lexer.  This is required.
    from calclex import tokens

    def p_expression_plus(p):
        'expression : expression PLUS term'
        p[0] = p[1] + p[3]

    def p_expression_minus(p):
        'expression : expression MINUS term'
        p[0] = p[1] - p[3]

    def p_expression_term(p):
        'expression : term'
        p[0] = p[1]

    def p_term_times(p):
        'term : term TIMES factor'
        p[0] = p[1] * p[3]

    def p_term_div(p):
        'term : term DIVIDE factor'
        p[0] = p[1] / p[3]

    def p_term_factor(p):
        'term : factor'
        p[0] = p[1]

    def p_factor_num(p):
        'factor : NUMBER'
        p[0] = p[1]

    def p_factor_expr(p):
        'factor : LPAREN expression RPAREN'
        p[0] = p[2]

    # Error rule for syntax errors
    def p_error(p):
        print("Syntax error in input!")

    # Build the parser
    parser = yacc.yacc()

    while True:
       try:
           s = input('calc > ')
       except EOFError:
           break
       if not s: continue
       result = parser.parse(s)
       print(result)

注意：“calclex.py”文件可在以下网址获取：
https://github.com/a200638012/ply/blob/master/tests/calclex.py（这个文件被更新了，内容有区别）


在该示例中，每个语法规则均由一个 Python 函数来定义，该函数的文档字符串包含了相应的无上下文语
法规范说明。构成函数主体的语句实现了该规则的语义操作。每个函数都接受一个名为 `p` 的单一参
数，该参数是一个序列，其中包含了对应于该规则中的每个语法符号的值。`p[i]` 的值会按照如下方
式映射到语法符号上：

    def p_expression_plus(p):
        'expression : expression PLUS term'
        #   ^            ^        ^    ^
        #  p[0]         p[1]     p[2] p[3]

        p[0] = p[1] + p[3]

对于标记符，其对应的 `p[i]` 的“值”与在词法解析模块中赋给 `p.value` 的属性完全相同。
对于非终结符，其值由规则简化时放入 `p[0]` 中的内容决定。这个值可以是任何东西。然而，
通常情况下，其值更可能是简单的 Python 类型、元组或实例。在这个示例中，我们利用了 `NUMBER` 
标记符在其值字段中存储整数值这一事实。其他所有规则都执行各种类型的整数运算并传播结果。

注意：在 yacc 中，负指数的使用具有特殊含义——特别是在此示例中，`p[-1]` 的值与 `p[3]` 
并不相同。有关更多详细信息，请参阅“嵌入式操作”部分。

yacc 规范中定义的第一条规则确定了起始语法符号（在本例中，`expression` 这条规则排在首位）。
每当解析器对起始规则进行归约且没有更多输入时，解析过程就会停止，并返回最终值（该值将是 `p[0]` 
中放置的最顶层规则的任何值）。注意：可以使用 `yacc()` 函数的 `start` 关键字参数来指定替代
的起始符号。

“p_error(p)”这一规则的定义目的是用于捕获语法错误。有关更多详细信息，请参阅下面的错误处理部分。

要构建解析器，请调用 `yacc.yacc()` 函数。该函数会查看模块，并尝试根据您指定的语
法构建所有的 LR 解析表。

如果在您的语法规范中发现任何错误，`yacc.py` 将会给出诊断信息，并可能引发异常。能够检测到的一
些错误包括：

-   重复的函数名称（如果在语法文件中存在多个规则函数具有相同的名称）。
-   由不明确的语法所引发的“移位/归约”和“归约/归约”冲突。
-   不明确的语法规则。
-   无限递归（永远无法终止的规则）。
-   未使用的规则和标记符。
-   未定义的规则和标记。

接下来的几个部分将更详细地探讨语法规范的相关内容。

该示例的最后一部分展示了如何实际运行由 `yacc()` 创建的解析器。要运行解析器，您需要调
用 `parse()` 函数，并传入一段输入文本字符串。这将执行所有的语法规则，并返回整个解析的
结果。这个结果的返回值就是赋给起始语法规则中的 `p[0]` 的值。

### 合并语法规则的功能

当语法规则相似时，它们可以合并为一个单一的功能。例如，参照我们之前示例中的两条规则：

    def p_expression_plus(p):
        'expression : expression PLUS term'
        p[0] = p[1] + p[3]

    def p_expression_minus(p):
        'expression : expression MINUS term'
        p[0] = p[1] - p[3]

与其编写两个函数，不如编写一个这样的单一函数：

    def p_expression(p):
        '''expression : expression PLUS term
                      | expression MINUS term'''
        if p[2] == '+':
            p[0] = p[1] + p[3]
        elif p[2] == '-':
            p[0] = p[1] - p[3]

一般来说，任何给定函数的文档字符串都可以包含多个语法规则。因此，也可以这样写：

    def p_binary_operators(p):
        '''expression : expression PLUS term
                      | expression MINUS term
           term       : term TIMES factor
                      | term DIVIDE factor'''
        if p[2] == '+':
            p[0] = p[1] + p[3]
        elif p[2] == '-':
            p[0] = p[1] - p[3]
        elif p[2] == '*':
            p[0] = p[1] * p[3]
        elif p[2] == '/':
            p[0] = p[1] / p[3]

在将语法规则整合到一个单一的函数中时，通常最好让所有规则具有相似的结构（例如，相同的项数）。
否则，相应的操作代码可能会比实际所需的更加复杂。不过，对于简单的情况，使用“len()”也是可行的。例如：

    def p_expressions(p):
        '''expression : expression MINUS expression
                      | MINUS expression'''
        if (len(p) == 4):
            p[0] = p[1] - p[3]
        elif (len(p) == 3):
            p[0] = -p[2]

如果对解析性能有要求，那么就应当避免在单个语法规则中加入过多的条件处理，就像上述示例中那
样。当您添加检查以确定正在处理哪条语法规则时，实际上您是在重复解析器已经完成的工作（即，
解析器已经确切地知道它匹配的是哪条规则）。通过为每条语法规则使用单独的 `p_rule()` 函数，
您可以消除这种额外开销。

### Character Literals

如果需要的话，语法中可以包含被定义为单个字符常量的标记。例如：

    def p_binary_operators(p):
        '''expression : expression '+' term
                      | expression '-' term
           term       : term '*' factor
                      | term '/' factor'''
        if p[2] == '+':
            p[0] = p[1] + p[3]
        elif p[2] == '-':
            p[0] = p[1] - p[3]
        elif p[2] == '*':
            p[0] = p[1] * p[3]
        elif p[2] == '/':
            p[0] = p[1] / p[3]

一个字符常量必须用引号括起来，例如 `'+'` 。此外，如果使用了字面值，那么必须通
过在相应的“lex”文件中使用特殊的“literals”声明来对其进行定义：

    # Literals should be placed in module given to lex()
    literals = ['+','-','*','/']
注意：请确保您未定义类似的重复令牌规则，例如 `t_...` ，否则将无法正常运行。

字符常量仅限于单个字符。因此，不能指定诸如 ``<=`` 或 ``==``这样的字符常量。对此，应使用常规
的词法解析规则（例如，定义一个规则如 `t_EQ = r'=='`）。


### 空语句处理

`yacc.py` 能够处理空语句项，其方法是定义这样一个规则：

    def p_empty(p):
        'empty :'
        pass

现在要使用这个空的生产规则，就用“empty”作为符号即可。例如：

    def p_optitem(p):
        'optitem : item'
        '        | empty'
        ...

注意：您可以在任何位置编写空规则，只需在右侧指定一个空值即可。不过，我个人认为编写一个“空”
规则并用“空”来表示空的产生式会更易于阅读，并且能更清晰地表达您的意图。

### 更改起始符号

通常，在 yacc 规范中首先出现的规则会定义起始语法规则（顶层规则）。若要更改此规则，可在
您的文件中添加一个“start”说明符。例如：

    start = 'foo'

    def p_bar(p):
        'bar : A B'

    # This is the starting rule due to the start specifier above
    def p_foo(p):
        'foo : bar X'
    ...

在调试过程中使用“start”指定符可能会很有用，因为你可以利用它来让 yacc 构建一个更大的语
法的一部分。为此，还可以将起始符号作为参数传递给“yacc()”函数来指定。例如：

    parser = yacc.yacc(start='foo')

### 应对模糊语法的情况

前面示例中给出的表达式语法是以一种特殊的格式书写而成，目的是为了消除歧义。然而，在许多情
况下，以这种格式来书写语法是非常困难或不自然的。更自然的表达语法的方式是采用更简洁的形式，
比如这样：

    expression : expression PLUS expression
               | expression MINUS expression
               | expression TIMES expression
               | expression DIVIDE expression
               | LPAREN expression RPAREN
               | NUMBER

不幸的是，这个语法规范存在歧义。例如，如果要解析字符串“3 * 4 + 5”，就无法确定运算符应
如何进行分组。比如，这个表达式到底是“((3 * 4) + 5)”还是“3 * (4 + 5)”呢？

如果给 `yacc.py` 传递了不明确的语法，它将会打印出有关“移位/归约冲突”或“归约/归约冲突”
的信息。移位/归约冲突是指解析器生成器无法决定是归约一个规则还是在解析栈上移位一个符号时
出现的情况。例如，考虑字符串“3 * 4 + 5”以及内部的解析栈：

    Step Symbol Stack           Input Tokens            Action
    ---- ---------------------  ---------------------   -------------------------------
    1    $                                3 * 4 + 5$    Shift 3
    2    $ 3                                * 4 + 5$    Reduce : expression : NUMBER
    3    $ expr                             * 4 + 5$    Shift *
    4    $ expr *                             4 + 5$    Shift 4
    5    $ expr * 4                             + 5$    Reduce: expression : NUMBER
    6    $ expr * expr                          + 5$    SHIFT/REDUCE CONFLICT ????

在这种情况下，当解析器到达步骤 6 时，它有两种选择。一种是将栈中的规则“expr : expr * expr”进行归约。
另一种是将栈中的符号“+”移出。根据上下文无关语法的规则，这两种选择都是完全合法的。

默认情况下，所有的左移/归约冲突都会倾向于选择左移操作。因此，在上述例子中，解析器总是会将`+`进行左移操作
，而不是进行归约操作。尽管这种策略在很多情况下都能奏效（例如，“如果-则”与“如果-则-否则”的情况），但对于
算术表达式来说是不够的。实际上，在上述例子中，将`+`进行左移的决策是完全错误的——我们应该将`expr * expr`
进行归约，因为乘法的数学优先级高于加法。

为了解决歧义问题，尤其是在表达式语法中，`yacc.py` 允许为单个标记指定优先级级别和结合性。这是通过在语法
文件中添加一个名为 `precedence` 的变量来实现的，其添加方式如下：

    precedence = (
        ('left', 'PLUS', 'MINUS'),
        ('left', 'TIMES', 'DIVIDE'),
    )

此声明明确指出，`PLUS`/`MINUS`具有相同的优先级级别，并且是左结合的；而`TIMES`/`DIVIDE`具有
相同的优先级，并且也是左结合的。在 `precedence`声明中，按照从低到高的优先级顺序对标记进行排序。
因此，此声明规定 `TIMES`/`DIVIDE`的优先级高于`PLUS`/`MINUS`（因为它们在优先级规范中出现得更
靠后）。

优先级规范的工作原理是为列出的标记赋予一个数值形式的优先级级别值以及一种结合方式。例如，在上述示
例中，您将得到：

    PLUS      : level = 1,  assoc = 'left'
    MINUS     : level = 1,  assoc = 'left'
    TIMES     : level = 2,  assoc = 'left'
    DIVIDE    : level = 2,  assoc = 'left'

然后，这些值会被用于为每个语法规则附加一个数值优先级值和结合性方向。*这总是通过查看最右侧
终结符的优先级来确定的。* 例如：

    expression : expression PLUS expression                 # level = 1, left
               | expression MINUS expression                # level = 1, left
               | expression TIMES expression                # level = 2, left
               | expression DIVIDE expression               # level = 2, left
               | LPAREN expression RPAREN                   # level = None (not specified)
               | NUMBER                                     # level = None (not specified)

当遇到移位/归约冲突时，解析器生成器会通过参考优先级规则和结合性说明来解决这一冲突。

Yacc 符号的优先级和结合性：

1.  如果当前的符号具有比栈中规则更高的优先级，那么该标记就会被移除。
2.  如果栈中的语法规则具有更高的优先级，则该规则会被归约。
3.  如果当前的符号和语法规则具有相同的优先级，那么对于左结合性，该规则将被简化；而对于右
    结合性，则该标记将被归约。
4.  如果对优先级关系一无所知，移位/归约 冲突则会优先考虑进行移位操作（这是默认设置）。

例如，如果 \"expression PLUS expression\" 已被解析，并且下一个标记是\"TIMES\"，那么操作将会是“移位”，
因为\"TIMES\"的优先级高于\"PLUS\"。另一方面，如果\"expression TIMES expression\"已被解析，并且下一个
标记是\"PLUS\"，那么操作将会是\"PLUS\"，因为“加法”的优先级低于\"TIMES\"。

当通过前三种方法（借助优先级规则）解决转换/归约冲突后，`yacc.py` 将不会报告语法中的任何错误或冲突
（尽管它会在 `parser.out` 调试文件中打印一些信息）。

优先级指定技术的一个问题在于，在某些情况下有时需要改变某个运算符的优先级。例如，在表达式\"3 + 4 \* -5\"中，
考虑一下一元减号运算符。从数学角度来看，一元减号通常具有非常高的优先级\--会先于乘法进行计算。然而，在我们的
优先级指定中，减号的优先级低于乘法。为了解决这个问题，可以为诸如这样的“虚构标记”设定优先级规则：

    precedence = (
        ('left', 'PLUS', 'MINUS'),
        ('left', 'TIMES', 'DIVIDE'),
        ('right', 'UMINUS'),            # Unary minus operator
    )

现在，在语法文件中，我们可以这样书写一元减号规则：

    def p_expr_uminus(p):
        'expression : MINUS expression %prec UMINUS'
        p[0] = -p[2]

在这种情况下，`%prec UMINUS` 会覆盖默认的规则优先级\--将其设置为优先级说明符中 UMINUS 的优先级。

首先，在这个示例中使用 UMINUS 可能会让人感到非常困惑。UMINUS 并非输入标记或语法规则。相反，您应该
将其视为优先级表中一个特殊标记的名称。当您使用 `%prec` 限定符时，您是在告诉 yacc 您希望表达式的优
先级与这个特殊标记的优先级相同，而不是通常的优先级。

还可以在`precedence`表中指定非结合性。这种设置适用于您不想让操作相互串联的情况。例如，假设您希望支持像“<”和“>”
这样的比较运算符，但又不想允许像“a < b < c”这样的组合。要实现这一点，可以设定这样的规则：

    precedence = (
        ('nonassoc', 'LESSTHAN', 'GREATERTHAN'),  # Nonassociative operators
        ('left', 'PLUS', 'MINUS'),
        ('left', 'TIMES', 'DIVIDE'),
        ('right', 'UMINUS'),            # Unary minus operator
    )

如果您这样做，输入文本如`a < b < c`就会导致语法错误。但像`a < b`这样的简单表达式则仍不会出错。

当针对一组符号存在多种可适用的语法规则时，就会产生“归约/归约”冲突。这种冲突几乎总是不好的，并且总
是通过选择语法文件中出现最早的那个规则来解决。归约/归约冲突几乎总是由不同的语法规则集以某种方式生
成相同的符号集而引起的。例如：

    assignment :  ID EQUALS NUMBER
               |  ID EQUALS expression

    expression : expression PLUS expression
               | expression MINUS expression
               | expression TIMES expression
               | expression DIVIDE expression
               | LPAREN expression RPAREN
               | NUMBER

在这种情况下，这两条规则之间存在“归约/归约”的冲突：

    assignment  : ID EQUALS NUMBER
    expression  : NUMBER

例如，如果你写的是\"a = 5\"，解析器无法确定这到底是应该被简化为`assignment : ID EQUALS NUMBER`
这种形式，还是应该先将 5 转换为一个表达式，然后再简化`assignment : ID EQUALS expression`这一规则。

It should be noted that reduce/reduce conflicts are notoriously
difficult to spot looking at the input grammar. When a reduce/reduce
conflict occurs, `yacc()` will try to help by printing a warning message
such as this:
需要指出的是，从输入语法来看，“归约/归约”冲突通常很难被发现。当出现“归约/归约”冲突时，`yacc()` 
会尝试通过打印一条警告信息来提供帮助，例如这样的信息：

    WARNING: 1 reduce/reduce conflict
    WARNING: reduce/reduce conflict in state 15 resolved using rule (assignment -> ID EQUALS NUMBER)
    WARNING: rejected rule (expression -> NUMBER)

此消息指出了存在冲突的两条规则。然而，它可能无法告知您解析器是如何达到这种状态的。
要尝试弄清楚这一点，您可能需要查看您的语法以及“parser.out”调试文件的内容，
这个工作可能会非常耗时。

### parser.out文件

找出“移位/归约”和“归约/归约”冲突是使用 LR 解析算法时的一大乐趣所在。为了便于调试，
`yacc.py` 可以生成一个名为“parser.out”的调试文件。要创建此文件，请使用 `yacc.yacc(debug=True)`
进行操作。该文件的内容大致如下：


    Unused terminals:


    Grammar

    Rule 1     expression -> expression PLUS expression
    Rule 2     expression -> expression MINUS expression
    Rule 3     expression -> expression TIMES expression
    Rule 4     expression -> expression DIVIDE expression
    Rule 5     expression -> NUMBER
    Rule 6     expression -> LPAREN expression RPAREN

    Terminals, with rules where they appear

    TIMES                : 3
    error                : 
    MINUS                : 2
    RPAREN               : 6
    LPAREN               : 6
    DIVIDE               : 4
    PLUS                 : 1
    NUMBER               : 5

    Nonterminals, with rules where they appear

    expression           : 1 1 2 2 3 3 4 4 6 0


    Parsing method: LALR


    state 0

        S' -> . expression
        expression -> . expression PLUS expression
        expression -> . expression MINUS expression
        expression -> . expression TIMES expression
        expression -> . expression DIVIDE expression
        expression -> . NUMBER
        expression -> . LPAREN expression RPAREN

        NUMBER          shift and go to state 3
        LPAREN          shift and go to state 2


    state 1

        S' -> expression .
        expression -> expression . PLUS expression
        expression -> expression . MINUS expression
        expression -> expression . TIMES expression
        expression -> expression . DIVIDE expression

        PLUS            shift and go to state 6
        MINUS           shift and go to state 5
        TIMES           shift and go to state 4
        DIVIDE          shift and go to state 7


    state 2

        expression -> LPAREN . expression RPAREN
        expression -> . expression PLUS expression
        expression -> . expression MINUS expression
        expression -> . expression TIMES expression
        expression -> . expression DIVIDE expression
        expression -> . NUMBER
        expression -> . LPAREN expression RPAREN

        NUMBER          shift and go to state 3
        LPAREN          shift and go to state 2


    state 3

        expression -> NUMBER .

        $               reduce using rule 5
        PLUS            reduce using rule 5
        MINUS           reduce using rule 5
        TIMES           reduce using rule 5
        DIVIDE          reduce using rule 5
        RPAREN          reduce using rule 5


    state 4

        expression -> expression TIMES . expression
        expression -> . expression PLUS expression
        expression -> . expression MINUS expression
        expression -> . expression TIMES expression
        expression -> . expression DIVIDE expression
        expression -> . NUMBER
        expression -> . LPAREN expression RPAREN

        NUMBER          shift and go to state 3
        LPAREN          shift and go to state 2


    state 5

        expression -> expression MINUS . expression
        expression -> . expression PLUS expression
        expression -> . expression MINUS expression
        expression -> . expression TIMES expression
        expression -> . expression DIVIDE expression
        expression -> . NUMBER
        expression -> . LPAREN expression RPAREN

        NUMBER          shift and go to state 3
        LPAREN          shift and go to state 2


    state 6

        expression -> expression PLUS . expression
        expression -> . expression PLUS expression
        expression -> . expression MINUS expression
        expression -> . expression TIMES expression
        expression -> . expression DIVIDE expression
        expression -> . NUMBER
        expression -> . LPAREN expression RPAREN

        NUMBER          shift and go to state 3
        LPAREN          shift and go to state 2


    state 7

        expression -> expression DIVIDE . expression
        expression -> . expression PLUS expression
        expression -> . expression MINUS expression
        expression -> . expression TIMES expression
        expression -> . expression DIVIDE expression
        expression -> . NUMBER
        expression -> . LPAREN expression RPAREN

        NUMBER          shift and go to state 3
        LPAREN          shift and go to state 2


    state 8

        expression -> LPAREN expression . RPAREN
        expression -> expression . PLUS expression
        expression -> expression . MINUS expression
        expression -> expression . TIMES expression
        expression -> expression . DIVIDE expression

        RPAREN          shift and go to state 13
        PLUS            shift and go to state 6
        MINUS           shift and go to state 5
        TIMES           shift and go to state 4
        DIVIDE          shift and go to state 7


    state 9

        expression -> expression TIMES expression .
        expression -> expression . PLUS expression
        expression -> expression . MINUS expression
        expression -> expression . TIMES expression
        expression -> expression . DIVIDE expression

        $               reduce using rule 3
        PLUS            reduce using rule 3
        MINUS           reduce using rule 3
        TIMES           reduce using rule 3
        DIVIDE          reduce using rule 3
        RPAREN          reduce using rule 3

      ! PLUS            [ shift and go to state 6 ]
      ! MINUS           [ shift and go to state 5 ]
      ! TIMES           [ shift and go to state 4 ]
      ! DIVIDE          [ shift and go to state 7 ]

    state 10

        expression -> expression MINUS expression .
        expression -> expression . PLUS expression
        expression -> expression . MINUS expression
        expression -> expression . TIMES expression
        expression -> expression . DIVIDE expression

        $               reduce using rule 2
        PLUS            reduce using rule 2
        MINUS           reduce using rule 2
        RPAREN          reduce using rule 2
        TIMES           shift and go to state 4
        DIVIDE          shift and go to state 7

      ! TIMES           [ reduce using rule 2 ]
      ! DIVIDE          [ reduce using rule 2 ]
      ! PLUS            [ shift and go to state 6 ]
      ! MINUS           [ shift and go to state 5 ]

    state 11

        expression -> expression PLUS expression .
        expression -> expression . PLUS expression
        expression -> expression . MINUS expression
        expression -> expression . TIMES expression
        expression -> expression . DIVIDE expression

        $               reduce using rule 1
        PLUS            reduce using rule 1
        MINUS           reduce using rule 1
        RPAREN          reduce using rule 1
        TIMES           shift and go to state 4
        DIVIDE          shift and go to state 7

      ! TIMES           [ reduce using rule 1 ]
      ! DIVIDE          [ reduce using rule 1 ]
      ! PLUS            [ shift and go to state 6 ]
      ! MINUS           [ shift and go to state 5 ]

    state 12

        expression -> expression DIVIDE expression .
        expression -> expression . PLUS expression
        expression -> expression . MINUS expression
        expression -> expression . TIMES expression
        expression -> expression . DIVIDE expression

        $               reduce using rule 4
        PLUS            reduce using rule 4
        MINUS           reduce using rule 4
        TIMES           reduce using rule 4
        DIVIDE          reduce using rule 4
        RPAREN          reduce using rule 4

      ! PLUS            [ shift and go to state 6 ]
      ! MINUS           [ shift and go to state 5 ]
      ! TIMES           [ shift and go to state 4 ]
      ! DIVIDE          [ shift and go to state 7 ]

    state 13

        expression -> LPAREN expression RPAREN .

        $               reduce using rule 6
        PLUS            reduce using rule 6
        MINUS           reduce using rule 6
        TIMES           reduce using rule 6
        DIVIDE          reduce using rule 6
        RPAREN          reduce using rule 6

此文件中出现的不同状态代表了该语法所允许的每一种有效的输入标记的可能序列。在接收输入标记
时，解析器会构建一个栈，并寻找匹配规则。每个状态都会记录此时可能正在匹配的语法规则。在每
个规则中，``.`` 字符表示解析在该规则中的当前位置。此外，还列出了每个有效输入标记的相应操作
。当出现移位/归约或归约/归约冲突时，未被选中的规则前缀会加上一个``!`` 符号。例如：

    ! TIMES           [ reduce using rule 2 ]
    ! DIVIDE          [ reduce using rule 2 ]
    ! PLUS            [ shift and go to state 6 ]
    ! MINUS           [ shift and go to state 5 ]

通过仔细研究这些规则（并稍加练习），通常就能找出大多数解析冲突的根源。此外，需要强调
的是，并非所有的移位-归约冲突都是不好的。然而，要确保这些冲突得到正确解决，唯一的办法
就是查看由``yacc.py``默认生成的``parser.out`` 文件，该文件可以通过向debug传递
``False``来禁用。

	yacc.yacc(debug=False)

### 语法错误处理

如果你正在为实际应用开发解析器，那么处理语法错误是非常重要的。一般来说，你不会希望解析器
在遇到第一个问题时就放弃并停止工作。相反，你希望它能够报告错误、尽可能地恢复，并继续进行
解析，以便一次性将输入中的所有错误都报告给用户。这是诸如 C、C++ 和 Java 等语言的编译器
所具有的标准行为。

在 PLY 中，如果在解析过程中出现语法错误，该错误会立即被检测到（即，解析器不会再读取
任何超出错误源范围的标记）。然而，在此阶段，解析器会进入一种恢复模式，可用于尝试继续
进行进一步的解析。一般来说，LR 解析器中的错误恢复是一个复杂的话题，涉及古老的仪式和
神秘的魔法。`yacc.py` 提供的恢复机制类似于 Unix 的 yacc，因此您可能需要参考像 
O'Reilly 的“Lex 和 Yacc”这样的书籍来了解一些更详细的细节。

当出现语法错误时，`yacc.py` 会执行以下步骤：

1.  在首次出现错误时，会调用用户自定义的 `p_error()` 函数，并将导致错误的标识符作为参数
    传递给它。然而，如果语法错误是由于到达文件末尾引起的，则 `p_error()` 函数会以 `None`
    作为参数进行调用。之后，解析器会进入“错误恢复”模式，在此模式下，它在成功将至少 3 个标识 
    符移到解析栈之前不会再次调用 `p_error()` 函数。
2.  如果在 `p_error()` 函数中未采取任何恢复措施，那么引发错误的前导标记将被替换为一个特殊的`error`标记。
3.  如果引发错误的前瞻标记已设置为`error`状态，那么解析栈中的最顶部元素就会被删除。
4.  如果整个解析栈被撤销，解析器就会进入重新启动状态，并尝试从其初始状态开始进行解析。
5.  如果某条语法规则将`error`视为一个标记，那么它就会被移到解析栈中。
6.  如果解析栈的最顶层元素是`error`，那么后续的待处理标记将会被丢弃，直到解析器能够成功地移除一
    个新符号或者对涉及`error`的规则进行缩减操作。

#### 使用错误规则进行回复和重新同步

处理语法错误时最恰当的方法是编写包含`error`标记的语法规则。例如，假设您的语言有一个类似于这样的
打印语句的语法规则：

    def p_statement_print(p):
         'statement : PRINT expr SEMI'
         ...

考虑到可能存在表达不当的情况，您可以添加一条这样的语法规则：

    def p_statement_print_error(p):
         'statement : PRINT error SEMI'
         print("Syntax error in print statement. Bad expression")

在这种情况下，`error` 标记将匹配在遇到第一个分号之前可能出现的任何一组标记。一旦到达
分号，规则就会被调用，而 `error` 标记就会消失。

这种类型的恢复有时被称为解析器重新同步。`error`标记可视为任何不良输入文本的通配符，
而紧接在 `error` 之后的标记则充当同步标记。

需要特别注意的是，在错误规则中，`error` 标记通常不会出现在右侧的最后一个位置。例如：

    def p_statement_print_error(p):
        'statement : PRINT error'
        print("Syntax error in print statement. Bad expression")

这是因为一旦遇到第一个错误的标记，规则就会被简化——如果随后紧接着又出现更多错误标记，
那么就可能难以恢复了。

#### 紧急模式恢复

另一种错误恢复方案是进入紧急模式恢复状态，在此状态下会丢弃部分标记，直到解析器能够以某
种合理的方式进行恢复为止。

紧急模式恢复操作完全由 `p_error()` 函数来实现。例如，该函数会开始丢弃标记，直至遇
到闭合的 `}` 符号。然后，它会将解析器重置到初始状态：

    def p_error(p):
        print("Whoa. You are seriously hosed.")
        if not p:
            print("End of File!")
            return

        # Read ahead looking for a closing '}'
        while True:
            tok = parser.token()             # Get the next token
            if not tok or tok.type == 'RBRACE': 
                break
        parser.restart()

此功能会丢弃错误的标记，并告知解析器该错误是可忽略的：

    def p_error(p):
        if p:
             print("Syntax error at token", p.type)
             # Just discard the token and tell the parser it's okay.
             parser.errok()
        else:
             print("Syntax error at EOF")

关于这些方法的更多详细信息如下：
`parser.errok()`

:   这会重置解析器的状态，使其不再认为自己处于错误恢复模式。这样就能避免生成`error`标记，
    并会重置内部错误计数器，以便下一次出现语法错误时会再次调用 `p_error()` 函数。

`parser.token()`

:   这会返回输入流中的下一个标记。

`parser.restart()`.

:  这会清除整个解析栈，并将解析器重置至初始状态。

为了向解析器提供下一个前瞻标记，`p_error()` 函数可以返回一个标记。如果需要对特殊字符进
行同步处理，这样做可能会很有用。例如：

    def p_error(p):
        # Read ahead looking for a terminating ";"
        while True:
            tok = parser.token()             # Get the next token
            if not tok or tok.type == 'SEMI': break
        parser.errok()

        # Return SEMI to the parser as the next lookahead token
        return tok  

请记住，上述错误处理函数中，`parser`是通过`yacc()`创建的解析器的一个实例。您需要将此实
例保存在您的代码中的某个位置，以便在错误处理时能够引用它。

#### 从一个产品中报处错误信号

如有必要，可以通过手动触发`SyntaxError`异常来强制解析器进入错误恢复模式。具体做法是像这样抛出该异常：

    def p_production(p):
        'production : some production ...'
        raise SyntaxError

提高`SyntaxError`这一设置的效果与将最后一个移至解析栈上的符号实际上是一个语法错误的效果相同。
因此，当您执行此操作时，移至解析栈的最后一个符号会被从解析栈中弹出，并且当前的预览标记会
被设置为 `error` 标记。然后，解析器会进入错误恢复模式，在此模式下它会尝试应用那些可以接受 `error` 
标记的规则。从这一点开始的后续步骤与检测到语法错误并调用`p_error()`的效果完全相同。

手动设置错误的一个重要方面是，在这种情况下 `p_error()` 函数不会被调用。如果您需要发
出错误消息，请务必在引发 `SyntaxError` 的代码段中进行操作。

注意：PLY 的此功能旨在模拟 yacc 中 YYERROR 宏的行为。

#### 语法错误何时会被报告？

在大多数情况下，当在输入中检测到错误的输入标记时，yacc 会立即处理这些错误。但请注意，
yacc 可能会选择在先对一个或多个语法规则进行归约之后再进行错误处理。这种行为可能出乎意料，
但它与底层解析表中的特殊状态（称为“默认状态”）有关。默认状态是指这样的解析条件：无论输入中
接下来出现的是什么有效的标记，都会对相同的语法规则进行归约。对于这类状态，yacc 会选择直接
对语法规则进行归约，而不读取接下来的输入标记。如果接下来的标记是错误的，yacc 最终会读取它
并报告语法错误。这只是有点不同寻常，因为您可能会看到某些语法规则在语法错误之前立即触发。

通常情况下，带有默认状态的延迟错误报告是无害的（而且也有其他原因使得我们希望 PLY 以这种方
式运行）。然而，如果出于某种原因需要关闭这种行为，您可以像下面这样清除默认状态表：

    parser = yacc.yacc()
    parser.defaulted_states = {}

如果您的语法使用了如第 6.11 （哪里来的6.11？）节所述的嵌入式操作，则不建议禁用默认状态。

#### 关于错误处理的一般性说明

对于常规类型的语言而言，利用错误规则和重新同步字符来进行错误恢复可能是最可靠的技术。
这是因为你可以对语法进行调整，以便在某些易于恢复并继续解析的位置捕获错误。紧急模式下
的恢复实际上只在某些特定的应用场景中才有用，在这些场景中，你可能需要丢弃大量输入文本
的一部分，以找到一个有效的重新开始点。

### 行号和位置跟踪

在编写编译器时，位置跟踪往往是一个颇具挑战性的问题。默认情况下，PLY 会记录所有标记的行
号和位置。可通过以下函数获取这些信息：

`p.lineno(num)`。返回符号 *num* 所在的行号。

`p.lexpos(num)`。返回符号 *num* 的词法位置。

例如：

    def p_expression(p):
        'expression : expression PLUS expression'
        line   = p.lineno(2)        # line number of the PLUS token
        index  = p.lexpos(2)        # Position of the PLUS token

作为一种可选功能，`yacc.py` 能够自动为所有的语法符号记录行号和位置信息。然而，这种额外的
跟踪需要额外的处理过程，并且会显著降低解析速度。因此，必须通过向 `yacc.parse()` 传递 
`tracking=True` 选项来启用此功能。例如：

    yacc.parse(data,tracking=True)

一旦启用，`lineno()` 和 `lexpos()` 方法将适用于所有语法符号。此外，还可以使用另外两个方法：

`p.linespan(num)`。返回一个元组 (起始行号，结束行号)，其中包含了符号 *num* 所在的起始行号和结束行号。

`p.lexspan(num)`。返回一个元组 (起始位置，结束位置)，其中包含符号 *num* 的起始位置和结束位置。

例如：

    def p_expression(p):
        'expression : expression PLUS expression'
        p.lineno(1)        # Line number of the left expression
        p.lineno(2)        # line number of the PLUS operator
        p.lineno(3)        # line number of the right expression
        ...
        start,end = p.linespan(3)    # Start,end lines of the right expression
        starti,endi = p.lexspan(3)   # Start,end positions of right expression

注意：`lexspan()` 函数仅返回至最后一个语法符号起始位置之前的数值范围。

虽然对于 PLY 来说，追踪所有语法符号的位置信息可能比较方便，但这种情况往往并非必要。例如，
如果只是在错误消息中使用行号信息，通常您只需依据语法规则中的某个特定标记即可。例如：

    def p_bad_func(p):
        'funccall : fname LPAREN error RPAREN'
        # Line number reported from LPAREN token
        print("Bad function call at line", p.lineno(2))

同样，如果您仅在需要的地方（通过使用 `p.set_lineno()` 方法）有选择地传播行号信息，
那么您可能会获得更好的解析性能。例如：

    def p_fname(p):
        'fname : ID'
        p[0] = p[1]
        p.set_lineno(0,p.lineno(1))

PLY 不会保留已解析规则的行号信息。如果您正在构建抽象语法树并且需要行号信息，那么您应当确
保行号信息直接出现在树中。

### 抽象语法树的构造

`yacc.py` 没有提供用于构建抽象语法树的特殊函数。不过，这种构建方式你自己也能很容易地完成。

构建一棵树的一种最简单的方法是在每个语法规则函数中创建并传播一个元组或列表。实现这一目标
的方法有很多，但以下就是一个示例：

    def p_expression_binop(p):
        '''expression : expression PLUS expression
                      | expression MINUS expression
                      | expression TIMES expression
                      | expression DIVIDE expression'''

        p[0] = ('binary-expression',p[2],p[1],p[3])

    def p_expression_group(p):
        'expression : LPAREN expression RPAREN'
        p[0] = ('group-expression',p[2])

    def p_expression_number(p):
        'expression : NUMBER'
        p[0] = ('number-expression',p[1])

另一种方法是为不同类型的抽象语法树节点创建一组数据结构，并在每个规则中将节点赋值给 `p[0]` 。例如：

    class Expr: pass

    class BinOp(Expr):
        def __init__(self,left,op,right):
            self.left = left
            self.right = right
            self.op = op

    class Number(Expr):
        def __init__(self,value):
            self.value = value

    def p_expression_binop(p):
        '''expression : expression PLUS expression
                      | expression MINUS expression
                      | expression TIMES expression
                      | expression DIVIDE expression'''

        p[0] = BinOp(p[1],p[2],p[3])

    def p_expression_group(p):
        'expression : LPAREN expression RPAREN'
        p[0] = p[2]

    def p_expression_number(p):
        'expression : NUMBER'
        p[0] = Number(p[1])

这种方法的优点在于，它可能使在节点类中附加更复杂的语义、类型检查、代码生成以及其他功能变得更加容易。

为了简化树的遍历过程，或许可以为你的解析树节点选择一种非常通用的树结构。例如：

    class Node:
        def __init__(self,type,children=None,leaf=None):
             self.type = type
             if children:
                  self.children = children
             else:
                  self.children = [ ]
             self.leaf = leaf

    def p_expression_binop(p):
        '''expression : expression PLUS expression
                      | expression MINUS expression
                      | expression TIMES expression
                      | expression DIVIDE expression'''

        p[0] = Node("binop", [p[1],p[3]], p[2])

### 嵌入式行为

yacc 所使用的解析技术仅允许在规则的末尾执行操作。例如，假设您有一个这样的规则：

    def p_foo(p):
        "foo : A B C D"
        print("Parsed a foo", p[1],p[2],p[3],p[4])

在这种情况下，所提供的操作代码只有在所有符号`A`, `B`, `C`,和 `D`都解析完毕后才会执行。
然而，有时在解析过程的中间阶段执行一小段代码片段也是很有用的。例如，假设您希望在解析完
 `A`之后立即执行某些操作。要实现这一点，可以编写一个如下的空规则：

    def p_foo(p):
        "foo : A seen_A B C D"
        print("Parsed a foo", p[1],p[3],p[4],p[5])
        print("seen_A returned", p[2])

    def p_seen_A(p):
        "seen_A :"
        print("Saw an A = ", p[-1])   # Access grammar symbol to left
        p[0] = some_value            # Assign value to seen_A

在该示例中，当`A`被推到解析栈上后，空的 `seen_A` 规则会立即执行。在此规则中， `p[-1]`
指的是位于`seen_A` 符号左侧紧邻的栈中的符号。在本例中，它将是`foo`规则上方紧邻的 `A` 的
值。与其他规则一样，可以通过将值赋给`p[0]`来从嵌入式操作中返回一个值。

嵌入式操作的使用有时可能会引发额外的移位/归约冲突。例如，以下这种语法就没有冲突情况：

    def p_foo(p):
        """foo : abcd
               | abcx"""

    def p_abcd(p):
        "abcd : A B C D"

    def p_abcx(p):
        "abcx : A B C X"

然而，如果您像这样在其中一条规则中插入一个嵌入式操作的话：

    def p_foo(p):
        """foo : abcd
               | abcx"""

    def p_abcd(p):
        "abcd : A B C D"

    def p_abcx(p):
        "abcx : A B seen_AB C X"

    def p_seen_AB(p):
        "seen_AB :"

将会引入一个额外的移位-归约冲突。这种冲突是由以下情况引起的：相同的符号 `C`在`abcd` 规
则和 `abcx`规则中紧挨着出现。解析器可以选择将该符号移位（使用`abcd`规则）或者归约空规
则`seen_AB` （使用`abcx`规则）。

嵌入式规则的一个常见用途是控制解析的其他方面，比如局部变量的作用域。例如，如果你正在解
析 C 代码，你可能会编写如下这样的代码：

    def p_statements_block(p):
        "statements: LBRACE new_scope statements RBRACE"""
        # Action code
        ...
        pop_scope()        # Return to previous scope

    def p_new_scope(p):
        "new_scope :"
        # Create a new scope for local variables
        s = new_scope()
        push_scope(s)
        ...

在这种情况下，嵌入式操作 `new_scope`会在解析到`LBRACE` (`{`)符号后立即执行。这可能会调整内部符号表
以及解析器的其他方面。在`statements_block`规则执行完毕后，代码可能会撤销嵌入式操作中执
行的那些操作（例如，调用`pop_scope()`函数）。

### 其它关于Yacc的注意事项

1.  默认情况下，`yacc.py` 依赖于 `lex.py` 来进行分词。不过，也可以像下面这样提供一个替代的分词器：

        parser = yacc.parse(lexer=x)

    在这种情况下，变量 `x` 必须是一个“解析器”对象，该对象至少要有一个用于获取下一个标记的 `x.token()` 
    方法。如果将输入字符串提供给 `yacc.parse()` 函数，那么解析器还必须具有 `x.input()` 方法。

2.  要在解析期间打印大量调试信息，请使用：

        parser.parse(input_text, debug=True)     

3.  由于 LR 分析是基于表格进行的，因此解析器的性能很大程度上不受语法大小的影响。最大的瓶颈将是词法分
    析器以及您语法规则中的代码复杂度。

4.  `yacc()` 还允许将解析器定义为类或闭包（请参阅关于对词法分析器进行替代性定义的章节）。但请注意，
    在单个模块（源文件）中只能定义一个解析器。如果在同一源文件中尝试定义多个解析器，可能会出现各种错
    误检查和验证步骤所引发的令人困惑的错误信息。

## 多个解析器和词法分析器

在高级解析应用中，您可能会需要多个解析器和词法分析器。

一般来说，这不会是个问题。但要使其正常运行，你需要仔细确保所有部件都正确连接。首先，要确保
保存住由 `lex()` 和 `yacc()` 返回的对象。例如：

    lexer  = lex.lex()       # Return lexer object
    parser = yacc.yacc()     # Return parser object

接下来，在解析过程中，请务必向 `parse()` 函数提供一个指向其应使用的词法分析器的引用。例如：

    parser.parse(text,lexer=lexer)

如果您忘了执行此操作，解析器将会使用最后创建的词法解析器——而这往往并非您所期望的结果。

在词法分析器和解析器的规则函数中，这些对象也是可用的。在词法分析器中，一个标记的“词法分析器”
属性指的是触发该规则的词法分析器对象。例如：

    def t_NUMBER(t):
       r'\d+'
       ...
       print(t.lexer)           ## Show lexer object

在解析器中，\"lexer\"和\"parser\"这两个属性分别指的是“词法分析器”对象和“解析器”对象：

    def p_expr_plus(p):
       'expr : expr PLUS expr'
       ...
       print(p.parser)          # Show parser object
       print(p.lexer)           # Show lexer object

如果需要的话，可以随意为词法分析器或解析器对象添加属性。例如，如果您想要设置不同的解析模式，
可以为解析器对象添加一个“模式”属性，并稍后对其进行查看。

## 高级调试

调试编译器通常并非易事。PLY 通过使用 Python 的 `logging` 模块提供了部分诊断功能。
接下来的两个部分将对此进行详细说明：

### 调试 lex() 和 yacc() 命令

 `lex()` 和 `yacc()` 这两个命令都具有一个调试模式，可通过使用`debug`标志来启用该模式。例如：

    lex.lex(debug=True)
    yacc.yacc(debug=True)

通常情况下，调试产生的输出会被定向到标准错误输出，或者（对于 `yacc()` 函数而言）被输出到一个名为
`parser.out` 的文件中。若要更精细地控制这些输出，可以通过提供一个日志对象来实现。下面是一个示例，
它会添加有关不同调试消息来源位置的信息：

    # Set up a logging object
    import logging
    logging.basicConfig(
        level = logging.DEBUG,
        filename = "parselog.txt",
        filemode = "w",
        format = "%(filename)10s:%(lineno)4d:%(message)s"
    )
    log = logging.getLogger()

    lex.lex(debug=True,debuglog=log)
    yacc.yacc(debug=True,debuglog=log)

如果您提供了自定义的日志记录器，那么生成的调试信息量可以通过设置日志级别来控制。
通常，调试消息会在`DEBUG`、`INFO`或`WARNING`级别下发出。

PLY 的错误消息和警告信息也是通过日志接口生成的。可以通过使用`errorlog`参数传递
一个日志对象来对其进行控制：

    lex.lex(errorlog=log)
    yacc.yacc(errorlog=log)

如果您希望完全屏蔽警告信息，您可以选择传入一个具有适当过滤级别的日志对象，或者使用在
`lex` 或 `yacc` 中定义的 `NullLogger` 对象。例如：

    yacc.yacc(errorlog=yacc.NullLogger())

### 运行时调试

若要实现解析器的运行时调试功能，请使用`debug`选项进行解析操作。此选项既可以是一个整数
（用于开启或关闭调试功能），也可以是一个日志记录对象的实例。例如：

    log = logging.getLogger()
    parser.parse(input,debug=log)

如果传递了一个日志对象，您可以利用其过滤级别来控制生成的输出量。`INFO` 级别用于生成有
关规则简化的信息。`DEBUG` 级别会显示有关解析栈、标记移动以及其他详细信息的内容。`ERROR`
级别则会显示与解析错误相关的信息。

对于非常复杂的任务，您应当传入一个日志对象，该对象会将日志记录重定向到一个文件中。这样，
在执行完毕后，您就能更方便地查看输出内容了。

## 使用Python -OO 模式

由于 PLY 依赖于文档字符串，因此它不兼容解释器的 [-OO]{.title-ref} 模式（该模式
会删除文档字符串）。如果您想要支持此功能，您需要编写一个装饰器或其他工具来为函数附加
文档字符串。例如：

    def _(doc):
        def decorate(func):
            func.__doc__ = doc
            return func
        return decorate

    @_("assignment : expr PLUS expr")
    def p_assignment(p):
        ...

PLY 默认情况下并不提供这样的装饰器。

## 下一步如何做？

PLY 分发包中的“examples”目录包含了几个简单的示例。有关理论、底层实现细节以及 LR 
分析法的相关内容，请参考编译器方面的教科书。
