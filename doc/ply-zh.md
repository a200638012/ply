```C++
本文主要采用机翻译为主，但是通过了人工校对。 --- 20251023 

有一些关键词机翻不统一，这里进行统一说明翻译含义：
token  ： 符号
```

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

在构建主（？这里翻译不太对）正则表达式时，规则的添加顺序如下：

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

It is important to note that storing data in other attribute names is
*not* recommended. The `yacc.py` module only exposes the contents of the
`value` attribute. Thus, accessing other attributes may be unnecessarily
awkward. If you need to store multiple values on a token, assign a
tuple, dictionary, or instance to `value`.
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

Be advised that if you are ignoring many different kinds of text, you
may still want to use functions since these provide more precise control
over the order in which regular expressions are matched (i.e., functions
are matched in order of specification whereas strings are sorted by
regular expression length).
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

The characters given in `t_ignore` are not ignored when such characters
are part of other regular expression patterns. For example, if you had a
rule to capture quoted text, that pattern can include the ignored
characters (which will be captured in the normal way). The main purpose
of `t_ignore` is to ignore whitespace and other padding between the
tokens that you actually want to parse.

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

It\'s possible to write token functions that perform additional actions
when literals are matched. However, you\'ll need to set the token type
appropriately. For example:
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

The `t_error()` function is used to handle lexing errors that occur when
illegal characters are detected. In this case, the `t.value` attribute
contains the rest of the input string that has not been tokenized. In
the example, the error function was defined as follows:

    # Error handling rule
    def t_error(t):
        print("Illegal character '%s'" % t.value[0])
        t.lexer.skip(1)

In this case, we print the offending character and skip ahead one
character by calling `t.lexer.skip(1)`.

### 换行处理

The `t_eof()` function is used to handle an end-of-file (EOF) condition
in the input. As input, it receives a token type `'eof'` with the
`lineno` and `lexpos` attributes set appropriately. The main use of this
function is provide more input to the lexer so that it can continue to
parse. Here is an example of how this works:

    # EOF handling rule
    def t_eof(t):
        # Get more input (Example)
        more = input('... ')
        if more:
            t.lexer.input(more)
            return t.lexer.token()
        return None

The EOF function should return the next available token (by calling
`t.lexer.token())` or `None` to indicate no more data. Be aware that
setting more input with the `t.lexer.input()` method does NOT reset
the lexer state or the `lineno` attribute used for position tracking.
The `lexpos` attribute is reset so be aware of that if you\'re using it
in error reporting.

### Building and using the lexer

To build the lexer, the function `lex.lex()` is used. For example:

    lexer = lex.lex()

This function uses Python reflection (or introspection) to read the
regular expression rules out of the calling context and build the lexer.
Once the lexer has been built, two methods can be used to control the
lexer:

`lexer.input(data)`. Reset the lexer and store a new input string.

`lexer.token()`. Return the next token. Returns a special `LexToken`
instance on success or None if the end of the input text has been
reached.

### The \@TOKEN decorator

In some applications, you may want to define tokens as a series of more
complex regular expression rules. For example:

    digit            = r'([0-9])'
    nondigit         = r'([_A-Za-z])'
    identifier       = r'(' + nondigit + r'(' + digit + r'|' + nondigit + r')*)'        

    def t_ID(t):
        # want docstring to be identifier above. ?????
        ...

In this case, we want the regular expression rule for `ID` to be one of
the variables above. However, there is no way to directly specify this
using a normal documentation string. To solve this problem, you can use
the `@TOKEN` decorator. For example:

    from ply.lex import TOKEN

    @TOKEN(identifier)
    def t_ID(t):
        ...

This will attach `identifier` to the docstring for `t_ID()` allowing
`lex.py` to work normally. Naturally, you could use `@TOKEN` on all
functions as an alternative to using docstrings.

### Debugging

For the purpose of debugging, you can run `lex()` in a debugging mode as
follows:

    lexer = lex.lex(debug=True)

This will produce various sorts of debugging information including all
of the added rules, the master regular expressions used by the lexer,
and tokens generating during lexing.

In addition, `lex.py` comes with a simple main function which will
either tokenize input read from standard input or from a file specified
on the command line. To use it, put this in your lexer:

    if __name__ == '__main__':
         lex.runmain()

Please refer to the \"Debugging\" section near the end for some more
advanced details of debugging.

### Alternative specification of lexers

As shown in the example, lexers are specified all within one Python
module. If you want to put token rules in a different module from the
one in which you invoke `lex()`, use the `module` keyword argument.

For example, you might have a dedicated module that just contains the
token rules:

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

Now, if you wanted to build a tokenizer from these rules from within a
different module, you would do the following (shown for Python
interactive mode):

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

The `module` option can also be used to define lexers from instances of
a class. For example:

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

When building a lexer from class, *you should construct the lexer from
an instance of the class*, not the class object itself. This is because
PLY only works properly if the lexer actions are defined by
bound-methods.

When using the `module` option to `lex()`, PLY collects symbols from the
underlying object using the `dir()` function. There is no direct access
to the `__dict__` attribute of the object supplied as a module value.

Finally, if you want to keep things nicely encapsulated, but don\'t want
to use a full-fledged class definition, lexers can be defined using
closures. For example:

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

Important note: If you are defining a lexer using a class or closure, be
aware that PLY still requires you to only define a single lexer per
module (source file). There are extensive validation/error checking
parts of the PLY that may falsely report error messages if you don\'t
follow this rule.

### Maintaining state

In your lexer, you may want to maintain a variety of state information.
This might include mode settings, symbol tables, and other details. As
an example, suppose that you wanted to keep track of how many NUMBER
tokens had been encountered.

One way to do this is to keep a set of global variables in the module
where you created the lexer. For example:

    num_count = 0
    def t_NUMBER(t):
        r'\d+'
        global num_count
        num_count += 1
        t.value = int(t.value)    
        return t

If you don\'t like the use of a global variable, another place to store
information is inside the Lexer object created by `lex()`. To do this, you
can use the `lexer` attribute of tokens passed to the various rules. For
example:

    def t_NUMBER(t):
        r'\d+'
        t.lexer.num_count += 1     # Note the use of lexer attribute
        t.value = int(t.value)    
        return t

    lexer = lex.lex()
    lexer.num_count = 0            # Set the initial count

This latter approach has the advantage of being simple and working
correctly in applications where multiple instantiations of a given lexer
exist in the same application. However, this might also feel like a
gross violation of encapsulation to OO purists. Just to put your mind at
some ease, all internal attributes of the lexer (with the exception of
`lineno`) have names that are prefixed by `lex` (e.g.,
`lexdata`, `lexpos`, etc.). Thus, it is perfectly safe to store
attributes in the lexer that don\'t have names starting with that prefix
or a name that conflicts with one of the predefined methods (e.g.,
`input()`, `token()`, etc.).

If you don\'t like assigning values on the lexer object, you can define
your lexer as a class as shown in the previous section:

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

The class approach may be the easiest to manage if your application is
going to be creating multiple instances of the same lexer and you need
to manage a lot of state.

State can also be managed through closures. For example:

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

### Lexer cloning

If necessary, a lexer object can be duplicated by invoking its `clone()`
method. For example:

    lexer = lex.lex()
    ...
    newlexer = lexer.clone()

When a lexer is cloned, the copy is exactly identical to the original
lexer including any input text and internal state. However, the clone
allows a different set of input text to be supplied which may be
processed separately. This may be useful in situations when you are
writing a parser/compiler that involves recursive or reentrant
processing. For instance, if you needed to scan ahead in the input for
some reason, you could create a clone and use it to look ahead. Or, if
you were implementing some kind of preprocessor, cloned lexers could be
used to handle different input files.

Creating a clone is different than calling `lex.lex()` in that PLY
doesn\'t regenerate any of the internal tables or regular expressions.

Special considerations need to be made when cloning lexers that also
maintain their own internal state using classes or closures. Namely, you
need to be aware that the newly created lexers will share all of this
state with the original lexer. For example, if you defined a lexer as a
class and did this:

    m = MyLexer()
    a = lex.lex(object=m)      # Create a lexer

    b = a.clone()              # Clone the lexer

Then both `a` and `b` are going to be bound to the same object `m` and
any changes to `m` will be reflected in both lexers. It\'s important to
emphasize that `clone()` is only meant to create a new lexer that reuses
the regular expressions and the environment of another lexer. If you need to
make a totally new copy of a lexer, then call `lex()` again.

### Internal lexer state

A Lexer object `lexer` has a number of internal attributes that may be
useful in certain situations:

`lexer.lexpos`

:   This attribute is an integer that contains the current position
    within the input text. If you modify the value, it will change the
    result of the next call to `token()`. Within token rule functions,
    this points to the first character *after* the matched text. If the
    value is modified within a rule, the next returned token will be
    matched at the new position.

`lexer.lineno`

:   The current value of the line number attribute stored in the lexer.
    PLY only specifies that the attribute exists\-\--it never sets,
    updates, or performs any processing with it. If you want to track
    line numbers, you will need to add code yourself (see the section on
    line numbers and positional information).

`lexer.lexdata`

:   The current input text stored in the lexer. This is the string
    passed with the `input()` method. It would probably be a bad idea to
    modify this unless you really know what you\'re doing.

`lexer.lexmatch`

:   This is the raw `Match` object returned by the Python `re.match()`
    function (used internally by PLY) for the current token. If you have
    written a regular expression that contains named groups, you can use
    this to retrieve those values.
	Note: This attribute is only updated when tokens are defined and processed by functions.

### Conditional lexing and start conditions

In advanced parsing applications, it may be useful to have different
lexing states. For instance, you may want the occurrence of a certain
token or syntactic construct to trigger a different kind of lexing. PLY
supports a feature that allows the underlying lexer to be put into a
series of different states. Each state can have its own tokens, lexing
rules, and so forth. The implementation is based largely on the \"start
condition\" feature of GNU flex. Details of this can be found at
<https://westes.github.io/flex/manual/Start-Conditions.html>

To define a new lexing state, it must first be declared. This is done by
including a \"states\" declaration in your lex file. For example:

    states = (
       ('foo','exclusive'),
       ('bar','inclusive'),
    )

This declaration declares two states, `'foo'` and `'bar'`. States may be
of two types; `'exclusive'` and `'inclusive'`. An ``'exclusive'`` state
completely overrides the default behavior of the lexer. That is, lex
will only return tokens and apply rules defined specifically for that
state. An ``'inclusive'`` state adds additional tokens and rules to the
default set of rules. Thus, lex will return both the tokens defined by
default in addition to those defined for the ``'inclusive'`` state.

Once a state has been declared, tokens and rules are declared by
including the state name in token/rule declaration. For example:

    t_foo_NUMBER = r'\d+'                      # Token 'NUMBER' in state 'foo'        
    t_bar_ID     = r'[a-zA-Z_][a-zA-Z0-9_]*'   # Token 'ID' in state 'bar'

    def t_foo_newline(t):
        r'\n'
        t.lexer.lineno += 1

A token can be declared in multiple states by including multiple state
names in the declaration. For example:

    t_foo_bar_NUMBER = r'\d+'         # Defines token 'NUMBER' in both state 'foo' and 'bar'

Alternative, a token can be declared in all states using the \'ANY\' in
the name:

    t_ANY_NUMBER = r'\d+'         # Defines a token 'NUMBER' in all states

If no state name is supplied, as is normally the case, the token is
associated with a special state `'INITIAL'`. For example, these two
declarations are identical:

    t_NUMBER = r'\d+'
    t_INITIAL_NUMBER = r'\d+'

States are also associated with the special `t_ignore`, `t_error()`, and
`t_eof()` declarations. For example, if a state treats these
differently, you can declare:

    t_foo_ignore = " \t\n"       # Ignored characters for state 'foo'

    def t_bar_error(t):          # Special error handler for state 'bar'
        pass 

By default, lexing operates in the `'INITIAL'` state. This state
includes all of the normally defined tokens. For users who aren\'t using
different states, this fact is completely transparent. If, during lexing
or parsing, you want to change the lexing state, use the `begin()`
method. For example:

    def t_begin_foo(t):
        r'start_foo'
        t.lexer.begin('foo')             # Starts 'foo' state

To get out of a state, you use `begin()` to switch back to the initial
state. For example:

    def t_foo_end(t):
        r'end_foo'
        t.lexer.begin('INITIAL')        # Back to the initial state

The management of states can also be done with a stack. For example:

    def t_begin_foo(t):
        r'start_foo'
        t.lexer.push_state('foo')             # Starts 'foo' state

    def t_foo_end(t):
        r'end_foo'
        t.lexer.pop_state()                   # Back to the previous state

The use of a stack would be useful in situations where there are many
ways of entering a new lexing state and you merely want to go back to
the previous state afterwards.

An example might help clarify. Suppose you were writing a parser and you
wanted to grab sections of arbitrary C code enclosed by curly braces.
That is, whenever you encounter a starting brace ``{``, you want to read
all of the enclosed code up to the ending brace ``}`` and return it as a
string. Doing this with a normal regular expression rule is nearly (if
not actually) impossible. This is because braces can be nested and can
be included in comments and strings. Thus, matching up to the first
matching ``}`` character isn\'t good enough. Here is how you might use
lexer states to do this:

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


In this example, the occurrence of the first ``{`` causes the lexer to
record the starting position and enter a new state `'ccode'`. A
collection of rules then match various parts of the input that follow
(comments, strings, etc.). All of these rules merely discard the token
(by not returning a value). However, if the closing right brace is
encountered, the rule `t_ccode_rbrace` collects all of the code (using
the earlier recorded starting position), stores it, and returns a token
\'CCODE\' containing all of that text. When returning the token, the
lexing state is restored back to its initial state.

### Miscellaneous Issues

-   The lexer requires input to be supplied as a single input string.
    Since most machines have more than enough memory, this rarely
    presents a performance concern. However, it means that the lexer
    currently can\'t be used with streaming data such as open files or
    sockets. This limitation is primarily a side-effect of using the
    `re` module. You might be able to work around this by implementing
    an appropriate `def t_eof()` end-of-file handling rule. The main
    complication here is that you\'ll probably need to ensure that data
    is fed to the lexer in a way so that it doesn\'t split in the
    middle of a token.

-   If you need to supply optional flags to the ``re.compile()`` function,
    supply the ``reflags`` option to lex. For example:

        lex.lex(reflags=re.UNICODE | re.VERBOSE)

    Note: by default, `reflags` is set to `re.VERBOSE`. If you provide
    your own flags, you may need to include this for PLY to preserve its
    normal behavior.

-   If you are going to create a hand-written lexer and you plan to use
    it with `yacc.py`, it only needs to conform to the following
    requirements:

    1.  It must provide a `token()` method that returns the next token
        or `None` if no more tokens are available.
    2.  The `token()` method must return an object `tok` that has `type`
        and `value` attributes. If line number tracking is being used,
        then the token should also define a `lineno` attribute.

## Parsing basics

`yacc.py` is used to parse language syntax. Before showing an example,
there are a few important bits of background that must be mentioned.
First, *syntax* is usually specified in terms of a BNF grammar. For
example, if you wanted to parse simple arithmetic expressions, you might
first write an unambiguous grammar specification like this:

    expression : expression + term
               | expression - term
               | term

    term       : term * factor
               | term / factor
               | factor

    factor     : NUMBER
               | ( expression )

In the grammar, symbols such as `NUMBER`, `+`, `-`, `*`, and `/` are
known as *terminals* and correspond to input tokens. Identifiers such as
`term` and `factor` refer to grammar rules comprised of a collection of
terminals and other rules. These identifiers are known as
*non-terminals*.

The semantic behavior of a language is often specified using a technique
known as syntax directed translation. In syntax directed translation,
attributes are attached to each symbol in a given grammar rule along
with an action. Whenever a particular grammar rule is recognized, the
action describes what to do. For example, given the expression grammar
above, you might write the specification for a simple calculator like
this:

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

A good way to think about syntax directed translation is to view each
symbol in the grammar as a kind of object. Associated with each symbol
is a value representing its \"state\" (for example, the `val` attribute
above). Semantic actions are then expressed as a collection of functions
or methods that operate on the symbols and associated values.

Yacc uses a parsing technique known as LR-parsing or shift-reduce
parsing. LR parsing is a bottom up technique that tries to recognize the
right-hand-side of various grammar rules. Whenever a valid
right-hand-side is found in the input, the appropriate action code is
triggered and the grammar symbols are replaced by the grammar symbol on
the left-hand-side.

LR parsing is commonly implemented by shifting grammar symbols onto a
stack and looking at the stack and the next input token for patterns
that match one of the grammar rules. The details of the algorithm can be
found in a compiler textbook, but the following example illustrates the
steps that are performed if you wanted to parse the expression
`3 + 5 * (10 - 20)` using the grammar defined above. In the example, the
special symbol `$` represents the end of input:

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

When parsing the expression, an underlying state machine and the current
input token determine what happens next. If the next token looks like
part of a valid grammar rule (based on other items on the stack), it is
generally shifted onto the stack. If the top of the stack contains a
valid right-hand-side of a grammar rule, it is usually \"reduced\" and
the symbols replaced with the symbol on the left-hand-side. When this
reduction occurs, the appropriate action is triggered (if defined). If
the input token can\'t be shifted and the top of stack doesn\'t match
any grammar rules, a syntax error has occurred and the parser must take
some kind of recovery step (or bail out). A parse is only successful if
the parser reaches a state where the symbol stack is empty and there are
no more input tokens.

It is important to note that the underlying implementation is built
around a large finite-state machine that is encoded in a collection of
tables. The construction of these tables is non-trivial and beyond the
scope of this discussion. However, subtle details of this process
explain why, in the example above, the parser chooses to shift a token
onto the stack in step 9 rather than reducing the rule
`expr : expr + term`.

## Yacc

The `ply.yacc` module implements the parsing component of PLY. The name
\"yacc\" stands for \"Yet Another Compiler Compiler\" and is borrowed
from the Unix tool of the same name.

### An example

Suppose you wanted to make a grammar for simple arithmetic expressions
as previously described. Here is how you would do it with `yacc.py`:

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

Note: ``calclex.py`` can be found at https://github.com/dabeaz/ply/blob/master/test/calclex.py

In this example, each grammar rule is defined by a Python function where
the docstring to that function contains the appropriate context-free
grammar specification. The statements that make up the function body
implement the semantic actions of the rule. Each function accepts a
single argument `p` that is a sequence containing the values of each
grammar symbol in the corresponding rule. The values of `p[i]` are
mapped to grammar symbols as shown here:

    def p_expression_plus(p):
        'expression : expression PLUS term'
        #   ^            ^        ^    ^
        #  p[0]         p[1]     p[2] p[3]

        p[0] = p[1] + p[3]

For tokens, the \"value\" of the corresponding `p[i]` is the *same* as
the `p.value` attribute assigned in the lexer module. For non-terminals,
the value is determined by whatever is placed in `p[0]` when rules are
reduced. This value can be anything at all. However, it probably most
common for the value to be a simple Python type, a tuple, or an
instance. In this example, we are relying on the fact that the `NUMBER`
token stores an integer value in its value field. All of the other rules
perform various types of integer operations and propagate the result.

Note: The use of negative indices have a special meaning in
yacc\-\--specially `p[-1]` does not have the same value as `p[3]` in
this example. Please see the section on \"Embedded Actions\" for further
details.

The first rule defined in the yacc specification determines the starting
grammar symbol (in this case, a rule for `expression` appears first).
Whenever the starting rule is reduced by the parser and no more input is
available, parsing stops and the final value is returned (this value
will be whatever the top-most rule placed in `p[0]`).
Note: an alternative starting symbol can be specified using the ``start`` keyword
argument to ``yacc()``.

The `p_error(p)` rule is defined to catch syntax errors. See the error
handling section below for more detail.

To build the parser, call the `yacc.yacc()` function. This function
looks at the module and attempts to construct all of the LR parsing
tables for the grammar you have specified.

If any errors are detected in your grammar specification, `yacc.py` will
produce diagnostic messages and possibly raise an exception. Some of the
errors that can be detected include:

-   Duplicated function names (if more than one rule function have the
    same name in the grammar file).
-   Shift/reduce and reduce/reduce conflicts generated by ambiguous
    grammars.
-   Badly specified grammar rules.
-   Infinite recursion (rules that can never terminate).
-   Unused rules and tokens
-   Undefined rules and tokens

The next few sections discuss grammar specification in more detail.

The final part of the example shows how to actually run the parser
created by `yacc()`. To run the parser, you have to call the `parse()`
with a string of input text. This will run all of the grammar rules and
return the result of the entire parse. This result return is the value
assigned to `p[0]` in the starting grammar rule.

### Combining Grammar Rule Functions

When grammar rules are similar, they can be combined into a single
function. For example, consider the two rules in our earlier example:

    def p_expression_plus(p):
        'expression : expression PLUS term'
        p[0] = p[1] + p[3]

    def p_expression_minus(p):
        'expression : expression MINUS term'
        p[0] = p[1] - p[3]

Instead of writing two functions, you might write a single function like
this:

    def p_expression(p):
        '''expression : expression PLUS term
                      | expression MINUS term'''
        if p[2] == '+':
            p[0] = p[1] + p[3]
        elif p[2] == '-':
            p[0] = p[1] - p[3]

In general, the docstring for any given function can contain multiple
grammar rules. So, it would have also been legal (although possibly
confusing) to write this:

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

When combining grammar rules into a single function, it is usually a
good idea for all of the rules to have a similar structure (e.g., the
same number of terms). Otherwise, the corresponding action code may be
more complicated than necessary. However, it is possible to handle
simple cases using ``len()``. For example:

    def p_expressions(p):
        '''expression : expression MINUS expression
                      | MINUS expression'''
        if (len(p) == 4):
            p[0] = p[1] - p[3]
        elif (len(p) == 3):
            p[0] = -p[2]

If parsing performance is a concern, you should resist the urge to put
too much conditional processing into a single grammar rule as shown in
these examples. When you add checks to see which grammar rule is being
handled, you are actually duplicating the work that the parser has
already performed (i.e., the parser already knows exactly what rule it
matched). You can eliminate this overhead by using a separate `p_rule()`
function for each grammar rule.

### Character Literals

If desired, a grammar may contain tokens defined as single character
literals. For example:

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

A character literal must be enclosed in quotes such as `'+'`. In
addition, if literals are used, they must be declared in the
corresponding `lex` file through the use of a special `literals`
declaration:

    # Literals should be placed in module given to lex()
    literals = ['+','-','*','/']
Note: make sure that you don't have a duplicate token rule defined like `t_...` to make it work.

Character literals are limited to a single character. Thus, it is not
legal to specify literals such as ``<=`` or ``==``. For this, use the
normal lexing rules (e.g., define a rule such as `t_EQ = r'=='`).

### Empty Productions

`yacc.py` can handle empty productions by defining a rule like this:

    def p_empty(p):
        'empty :'
        pass

Now to use the empty production, use ``empty`` as a symbol. For example:

    def p_optitem(p):
        'optitem : item'
        '        | empty'
        ...

Note: You can write empty rules anywhere by specifying an empty right
hand side. However, I personally find that writing an \"empty\" rule and
using \"empty\" to denote an empty production is easier to read and more
clearly states your intentions.

### Changing the starting symbol

Normally, the first rule found in a yacc specification defines the
starting grammar rule (top level rule). To change this, supply a `start`
specifier in your file. For example:

    start = 'foo'

    def p_bar(p):
        'bar : A B'

    # This is the starting rule due to the start specifier above
    def p_foo(p):
        'foo : bar X'
    ...

The use of a `start` specifier may be useful during debugging since you
can use it to have yacc build a subset of a larger grammar. For this
purpose, it is also possible to specify a starting symbol as an argument
to `yacc()`. For example:

    parser = yacc.yacc(start='foo')

### Dealing With Ambiguous Grammars

The expression grammar given in the earlier example has been written in
a special format to eliminate ambiguity. However, in many situations, it
is extremely difficult or awkward to write grammars in this format. A
much more natural way to express the grammar is in a more compact form
like this:

    expression : expression PLUS expression
               | expression MINUS expression
               | expression TIMES expression
               | expression DIVIDE expression
               | LPAREN expression RPAREN
               | NUMBER

Unfortunately, this grammar specification is ambiguous. For example, if
you are parsing the string \"3 \* 4 + 5\", there is no way to tell how
the operators are supposed to be grouped. For example, does the
expression mean \"(3 \* 4) + 5\" or is it \"3 \* (4+5)\"?

When an ambiguous grammar is given to `yacc.py` it will print messages
about \"shift/reduce conflicts\" or \"reduce/reduce conflicts\". A
shift/reduce conflict is caused when the parser generator can\'t decide
whether or not to reduce a rule or shift a symbol on the parsing stack.
For example, consider the string \"3 \* 4 + 5\" and the internal parsing
stack:

    Step Symbol Stack           Input Tokens            Action
    ---- ---------------------  ---------------------   -------------------------------
    1    $                                3 * 4 + 5$    Shift 3
    2    $ 3                                * 4 + 5$    Reduce : expression : NUMBER
    3    $ expr                             * 4 + 5$    Shift *
    4    $ expr *                             4 + 5$    Shift 4
    5    $ expr * 4                             + 5$    Reduce: expression : NUMBER
    6    $ expr * expr                          + 5$    SHIFT/REDUCE CONFLICT ????

In this case, when the parser reaches step 6, it has two options. One is
to reduce the rule `expr : expr * expr` on the stack. The other option
is to shift the token `+` on the stack. Both options are perfectly legal
from the rules of the context-free-grammar.

By default, all shift/reduce conflicts are resolved in favor of
shifting. Therefore, in the above example, the parser will always shift
the `+` instead of reducing. Although this strategy works in many cases
(for example, the case of \"if-then\" versus \"if-then-else\"), it is
not enough for arithmetic expressions. In fact, in the above example,
the decision to shift `+` is completely wrong\-\--we should have reduced
`expr * expr` since multiplication has higher mathematical precedence
than addition.

To resolve ambiguity, especially in expression grammars, `yacc.py`
allows individual tokens to be assigned a precedence level and
associativity. This is done by adding a variable `precedence` to the
grammar file like this:

    precedence = (
        ('left', 'PLUS', 'MINUS'),
        ('left', 'TIMES', 'DIVIDE'),
    )

This declaration specifies that `PLUS`/`MINUS` have the same precedence
level and are left-associative and that `TIMES`/`DIVIDE` have the same
precedence and are left-associative. Within the `precedence`
declaration, tokens are ordered from lowest to highest precedence. Thus,
this declaration specifies that `TIMES`/`DIVIDE` have higher precedence
than `PLUS`/`MINUS` (since they appear later in the precedence
specification).

The precedence specification works by associating a numerical precedence
level value and associativity direction to the listed tokens. For
example, in the above example you will get:

    PLUS      : level = 1,  assoc = 'left'
    MINUS     : level = 1,  assoc = 'left'
    TIMES     : level = 2,  assoc = 'left'
    DIVIDE    : level = 2,  assoc = 'left'

These values are then used to attach a numerical precedence value and
associativity direction to each grammar rule. *This is always determined
by looking at the precedence of the right-most terminal symbol.* For
example:

    expression : expression PLUS expression                 # level = 1, left
               | expression MINUS expression                # level = 1, left
               | expression TIMES expression                # level = 2, left
               | expression DIVIDE expression               # level = 2, left
               | LPAREN expression RPAREN                   # level = None (not specified)
               | NUMBER                                     # level = None (not specified)

When shift/reduce conflicts are encountered, the parser generator
resolves the conflict by looking at the precedence rules and
associativity specifiers.

Yacc precedence and associativity of tokens:

1.  If the current token has higher precedence than the rule on the
    stack, it is shifted.
2.  If the grammar rule on the stack has higher precedence, the rule is
    reduced.
3.  If the current token and the grammar rule have the same precedence,
    the rule is reduced for left associativity, whereas the token is
    shifted for right associativity.
4.  If nothing is known about the precedence, shift/reduce conflicts are
    resolved in favor of shifting (the default).

For example, if \"expression PLUS expression\" has been parsed and the
next token is \"TIMES\", the action is going to be a shift because
\"TIMES\" has a higher precedence level than \"PLUS\". On the other
hand, if \"expression TIMES expression\" has been parsed and the next
token is \"PLUS\", the action is going to be reduce because \"PLUS\" has
a lower precedence than \"TIMES.\"

When shift/reduce conflicts are resolved using the first three
techniques (with the help of precedence rules), `yacc.py` will report no
errors or conflicts in the grammar (although it will print some
information in the `parser.out` debugging file).

One problem with the precedence specifier technique is that it is
sometimes necessary to change the precedence of an operator in certain
contexts. For example, consider a unary-minus operator in \"3 + 4 \*
-5\". Mathematically, the unary minus is normally given a very high
precedence\--being evaluated before the multiply. However, in our
precedence specifier, MINUS has a lower precedence than TIMES. To deal
with this, precedence rules can be given for so-called \"fictitious
tokens\" like this:

    precedence = (
        ('left', 'PLUS', 'MINUS'),
        ('left', 'TIMES', 'DIVIDE'),
        ('right', 'UMINUS'),            # Unary minus operator
    )

Now, in the grammar file, we can write our unary minus rule like this:

    def p_expr_uminus(p):
        'expression : MINUS expression %prec UMINUS'
        p[0] = -p[2]

In this case, `%prec UMINUS` overrides the default rule
precedence\--setting it to that of UMINUS in the precedence specifier.

At first, the use of UMINUS in this example may appear very confusing.
UMINUS is not an input token or a grammar rule. Instead, you should
think of it as the name of a special marker in the precedence table.
When you use the `%prec` qualifier, you\'re telling yacc that you want
the precedence of the expression to be the same as for this special
marker instead of the usual precedence.

It is also possible to specify non-associativity in the `precedence`
table. This would be used when you *don\'t* want operations to chain
together. For example, suppose you wanted to support comparison
operators like `<` and `>` but you didn\'t want to allow combinations
like `a < b < c`. To do this, specify a rule like this:

    precedence = (
        ('nonassoc', 'LESSTHAN', 'GREATERTHAN'),  # Nonassociative operators
        ('left', 'PLUS', 'MINUS'),
        ('left', 'TIMES', 'DIVIDE'),
        ('right', 'UMINUS'),            # Unary minus operator
    )

If you do this, the occurrence of input text such as `a < b < c` will
result in a syntax error. However, simple expressions such as `a < b`
will still be fine.

Reduce/reduce conflicts are caused when there are multiple grammar rules
that can be applied to a given set of symbols. This kind of conflict is
almost always bad and is always resolved by picking the rule that
appears first in the grammar file. Reduce/reduce conflicts are almost
always caused when different sets of grammar rules somehow generate the
same set of symbols. For example:

    assignment :  ID EQUALS NUMBER
               |  ID EQUALS expression

    expression : expression PLUS expression
               | expression MINUS expression
               | expression TIMES expression
               | expression DIVIDE expression
               | LPAREN expression RPAREN
               | NUMBER

In this case, a reduce/reduce conflict exists between these two rules:

    assignment  : ID EQUALS NUMBER
    expression  : NUMBER

For example, if you wrote \"a = 5\", the parser can\'t figure out if
this is supposed to be reduced as `assignment : ID EQUALS NUMBER` or
whether it\'s supposed to reduce the 5 as an expression and then reduce
the rule `assignment : ID EQUALS expression`.

It should be noted that reduce/reduce conflicts are notoriously
difficult to spot looking at the input grammar. When a reduce/reduce
conflict occurs, `yacc()` will try to help by printing a warning message
such as this:

    WARNING: 1 reduce/reduce conflict
    WARNING: reduce/reduce conflict in state 15 resolved using rule (assignment -> ID EQUALS NUMBER)
    WARNING: rejected rule (expression -> NUMBER)

This message identifies the two rules that are in conflict. However, it
may not tell you how the parser arrived at such a state. To try and
figure it out, you\'ll probably have to look at your grammar and the
contents of the `parser.out` debugging file with an appropriately high
level of caffeination.

### The parser.out file

Tracking down shift/reduce and reduce/reduce conflicts is one of the
finer pleasures of using an LR parsing algorithm. To assist in
debugging, `yacc.py` can create a debugging file called \'parser.out\'.
To create this file, use `yacc.yacc(debug=True)`. The contents of this
file look like the following:

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

The different states that appear in this file are a representation of
every possible sequence of valid input tokens allowed by the grammar.
When receiving input tokens, the parser is building up a stack and
looking for matching rules. Each state keeps track of the grammar rules
that might be in the process of being matched at that point. Within each
rule, the ``.`` character indicates the current location of the parse
within that rule. In addition, the actions for each valid input token
are listed. When a shift/reduce or reduce/reduce conflict arises, rules
*not* selected are prefixed with an ``!``. For example:

    ! TIMES           [ reduce using rule 2 ]
    ! DIVIDE          [ reduce using rule 2 ]
    ! PLUS            [ shift and go to state 6 ]
    ! MINUS           [ shift and go to state 5 ]

By looking at these rules (and with a little practice), you can usually
track down the source of most parsing conflicts. It should also be
stressed that not all shift-reduce conflicts are bad. However, the only
way to be sure that they are resolved correctly is to look at
``parser.out`` file generated by ``yacc.py`` by default, can be disabled by passing ``False`` to debug::

	yacc.yacc(debug=False)

### Syntax Error Handling

If you are creating a parser for production use, the handling of syntax
errors is important. As a general rule, you don\'t want a parser to
throw up its hands and stop at the first sign of trouble. Instead, you
want it to report the error, recover if possible, and continue parsing
so that all of the errors in the input get reported to the user at once.
This is the standard behavior found in compilers for languages such as
C, C++, and Java.

In PLY, when a syntax error occurs during parsing, the error is
immediately detected (i.e., the parser does not read any more tokens
beyond the source of the error). However, at this point, the parser
enters a recovery mode that can be used to try and continue further
parsing. As a general rule, error recovery in LR parsers is a delicate
topic that involves ancient rituals and black-magic. The recovery
mechanism provided by `yacc.py` is comparable to Unix yacc so you may
want consult a book like O\'Reilly\'s \"Lex and Yacc\" for some of the
finer details.

When a syntax error occurs, `yacc.py` performs the following steps:

1.  On the first occurrence of an error, the user-defined `p_error()`
    function is called with the offending token as an argument. However,
    if the syntax error is due to reaching the end-of-file, `p_error()`
    is called with an argument of `None`. Afterwards, the parser enters
    an \"error-recovery\" mode in which it will not make future calls to
    `p_error()` until it has successfully shifted at least 3 tokens onto
    the parsing stack.
2.  If no recovery action is taken in `p_error()`, the offending
    lookahead token is replaced with a special `error` token.
3.  If the offending lookahead token is already set to `error`, the top
    item of the parsing stack is deleted.
4.  If the entire parsing stack is unwound, the parser enters a restart
    state and attempts to start parsing from its initial state.
5.  If a grammar rule accepts `error` as a token, it will be shifted
    onto the parsing stack.
6.  If the top item of the parsing stack is `error`, lookahead tokens
    will be discarded until the parser can successfully shift a new
    symbol or reduce a rule involving `error`.

#### Recovery and resynchronization with error rules

The most well-behaved approach for handling syntax errors is to write
grammar rules that include the `error` token. For example, suppose your
language had a grammar rule for a print statement like this:

    def p_statement_print(p):
         'statement : PRINT expr SEMI'
         ...

To account for the possibility of a bad expression, you might write an
additional grammar rule like this:

    def p_statement_print_error(p):
         'statement : PRINT error SEMI'
         print("Syntax error in print statement. Bad expression")

In this case, the `error` token will match any sequence of tokens that
might appear up to the first semicolon that is encountered. Once the
semicolon is reached, the rule will be invoked and the `error` token
will go away.

This type of recovery is sometimes known as parser resynchronization.
The `error` token acts as a wildcard for any bad input text and the
token immediately following `error` acts as a synchronization token.

It is important to note that the `error` token usually does not appear
as the last token on the right in an error rule. For example:

    def p_statement_print_error(p):
        'statement : PRINT error'
        print("Syntax error in print statement. Bad expression")

This is because the first bad token encountered will cause the rule to
be reduced\--which may make it difficult to recover if more bad tokens
immediately follow.

#### Panic mode recovery

An alternative error recovery scheme is to enter a panic mode recovery
in which tokens are discarded to a point where the parser might be able
to recover in some sensible manner.

Panic mode recovery is implemented entirely in the `p_error()` function.
For example, this function starts discarding tokens until it reaches a
closing \'}\'. Then, it restarts the parser in its initial state:

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

This function discards the bad token and tells the parser that the error
was ok:

    def p_error(p):
        if p:
             print("Syntax error at token", p.type)
             # Just discard the token and tell the parser it's okay.
             parser.errok()
        else:
             print("Syntax error at EOF")

More information on these methods is as follows:

`parser.errok()`

:   This resets the parser state so it doesn\'t think it\'s in
    error-recovery mode. This will prevent an `error` token from being
    generated and will reset the internal error counters so that the
    next syntax error will call `p_error()` again.

`parser.token()`

:   This returns the next token on the input stream.

`parser.restart()`.

:   This discards the entire parsing stack and resets the parser to its
    initial state.

To supply the next lookahead token to the parser, `p_error()` can return
a token. This might be useful if trying to synchronize on special
characters. For example:

    def p_error(p):
        # Read ahead looking for a terminating ";"
        while True:
            tok = parser.token()             # Get the next token
            if not tok or tok.type == 'SEMI': break
        parser.errok()

        # Return SEMI to the parser as the next lookahead token
        return tok  

Keep in mind in that the above error handling functions, `parser` is an
instance of the parser created by `yacc()`. You\'ll need to save this
instance someplace in your code so that you can refer to it during error
handling.

#### Signalling an error from a production

If necessary, a production rule can manually force the parser to enter
error recovery. This is done by raising the `SyntaxError` exception like
this:

    def p_production(p):
        'production : some production ...'
        raise SyntaxError

The effect of raising `SyntaxError` is the same as if the last symbol
shifted onto the parsing stack was actually a syntax error. Thus, when
you do this, the last symbol shifted is popped off of the parsing stack
and the current lookahead token is set to an `error` token. The parser
then enters error-recovery mode where it tries to reduce rules that can
accept `error` tokens. The steps that follow from this point are exactly
the same as if a syntax error were detected and `p_error()` were called.

One important aspect of manually setting an error is that the
`p_error()` function will NOT be called in this case. If you need to
issue an error message, make sure you do it in the production that
raises `SyntaxError`.

Note: This feature of PLY is meant to mimic the behavior of the YYERROR
macro in yacc.

#### When Do Syntax Errors Get Reported?

In most cases, yacc will handle errors as soon as a bad input token is
detected on the input. However, be aware that yacc may choose to delay
error handling until after it has reduced one or more grammar rules
first. This behavior might be unexpected, but it\'s related to special
states in the underlying parsing table known as \"defaulted states.\" A
defaulted state is parsing condition where the same grammar rule will be
reduced regardless of what *valid* token comes next on the input. For
such states, yacc chooses to go ahead and reduce the grammar rule
*without reading the next input token*. If the next token is bad, yacc
will eventually get around to reading it and report a syntax error.
It\'s just a little unusual in that you might see some of your grammar
rules firing immediately prior to the syntax error.

Usually, the delayed error reporting with defaulted states is harmless
(and there are other reasons for wanting PLY to behave in this way).
However, if you need to turn this behavior off for some reason. You can
clear the defaulted states table like this:

    parser = yacc.yacc()
    parser.defaulted_states = {}

Disabling defaulted states is not recommended if your grammar makes use
of embedded actions as described in Section 6.11.

#### General comments on error handling

For normal types of languages, error recovery with error rules and
resynchronization characters is probably the most reliable technique.
This is because you can instrument the grammar to catch errors at
selected places where it is relatively easy to recover and continue
parsing. Panic mode recovery is really only useful in certain
specialized applications where you might want to discard huge portions
of the input text to find a valid restart point.

### Line Number and Position Tracking

Position tracking is often a tricky problem when writing compilers. By
default, PLY tracks the line number and position of all tokens. This
information is available using the following functions:

`p.lineno(num)`. Return the line number for symbol *num*

`p.lexpos(num)`. Return the lexing position for symbol *num*

For example:

    def p_expression(p):
        'expression : expression PLUS expression'
        line   = p.lineno(2)        # line number of the PLUS token
        index  = p.lexpos(2)        # Position of the PLUS token

As an optional feature, `yacc.py` can automatically track line numbers
and positions for all of the grammar symbols as well. However, this
extra tracking requires extra processing and can significantly slow down
parsing. Therefore, it must be enabled by passing the `tracking=True`
option to `yacc.parse()`. For example:

    yacc.parse(data,tracking=True)

Once enabled, the `lineno()` and `lexpos()` methods work for all grammar
symbols. In addition, two additional methods can be used:

`p.linespan(num)`. Return a tuple (startline,endline) with the starting
and ending line number for symbol *num*.

`p.lexspan(num)`. Return a tuple (start,end) with the starting and
ending positions for symbol *num*.

For example:

    def p_expression(p):
        'expression : expression PLUS expression'
        p.lineno(1)        # Line number of the left expression
        p.lineno(2)        # line number of the PLUS operator
        p.lineno(3)        # line number of the right expression
        ...
        start,end = p.linespan(3)    # Start,end lines of the right expression
        starti,endi = p.lexspan(3)   # Start,end positions of right expression

Note: The `lexspan()` function only returns the range of values up to
the start of the last grammar symbol.

Although it may be convenient for PLY to track position information on
all grammar symbols, this is often unnecessary. For example, if you are
merely using line number information in an error message, you can often
just key off of a specific token in the grammar rule. For example:

    def p_bad_func(p):
        'funccall : fname LPAREN error RPAREN'
        # Line number reported from LPAREN token
        print("Bad function call at line", p.lineno(2))

Similarly, you may get better parsing performance if you only
selectively propagate line number information where it\'s needed using
the `p.set_lineno()` method. For example:

    def p_fname(p):
        'fname : ID'
        p[0] = p[1]
        p.set_lineno(0,p.lineno(1))

PLY doesn\'t retain line number information from rules that have already
been parsed. If you are building an abstract syntax tree and need to
have line numbers, you should make sure that the line numbers appear in
the tree itself.

### AST Construction

`yacc.py` provides no special functions for constructing an abstract
syntax tree. However, such construction is easy enough to do on your
own.

A minimal way to construct a tree is to create and propagate a tuple or
list in each grammar rule function. There are many possible ways to do
this, but one example would be something like this:

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

Another approach is to create a set of data structure for different
kinds of abstract syntax tree nodes and assign nodes to `p[0]` in each
rule. For example:

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

The advantage to this approach is that it may make it easier to attach
more complicated semantics, type checking, code generation, and other
features to the node classes.

To simplify tree traversal, it may make sense to pick a very generic
tree structure for your parse tree nodes. For example:

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

### Embedded Actions

The parsing technique used by yacc only allows actions to be executed at
the end of a rule. For example, suppose you have a rule like this:

    def p_foo(p):
        "foo : A B C D"
        print("Parsed a foo", p[1],p[2],p[3],p[4])

In this case, the supplied action code only executes after all of the
symbols `A`, `B`, `C`, and `D` have been parsed. Sometimes, however, it
is useful to execute small code fragments during intermediate stages of
parsing. For example, suppose you wanted to perform some action
immediately after `A` has been parsed. To do this, write an empty rule
like this:

    def p_foo(p):
        "foo : A seen_A B C D"
        print("Parsed a foo", p[1],p[3],p[4],p[5])
        print("seen_A returned", p[2])

    def p_seen_A(p):
        "seen_A :"
        print("Saw an A = ", p[-1])   # Access grammar symbol to left
        p[0] = some_value            # Assign value to seen_A

In this example, the empty `seen_A` rule executes immediately after `A`
is shifted onto the parsing stack. Within this rule, `p[-1]` refers to
the symbol on the stack that appears immediately to the left of the
`seen_A` symbol. In this case, it would be the value of `A` in the `foo`
rule immediately above. Like other rules, a value can be returned from
an embedded action by assigning it to `p[0]`

The use of embedded actions can sometimes introduce extra shift/reduce
conflicts. For example, this grammar has no conflicts:

    def p_foo(p):
        """foo : abcd
               | abcx"""

    def p_abcd(p):
        "abcd : A B C D"

    def p_abcx(p):
        "abcx : A B C X"

However, if you insert an embedded action into one of the rules like
this:

    def p_foo(p):
        """foo : abcd
               | abcx"""

    def p_abcd(p):
        "abcd : A B C D"

    def p_abcx(p):
        "abcx : A B seen_AB C X"

    def p_seen_AB(p):
        "seen_AB :"

an extra shift-reduce conflict will be introduced. This conflict is
caused by the fact that the same symbol `C` appears next in both the
`abcd` and `abcx` rules. The parser can either shift the symbol (`abcd`
rule) or reduce the empty rule `seen_AB` (`abcx` rule).

A common use of embedded rules is to control other aspects of parsing
such as scoping of local variables. For example, if you were parsing C
code, you might write code like this:

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

In this case, the embedded action `new_scope` executes immediately after
a `LBRACE` (`{`) symbol is parsed. This might adjust internal symbol
tables and other aspects of the parser. Upon completion of the rule
`statements_block`, code might undo the operations performed in the
embedded action (e.g., `pop_scope()`).

### Miscellaneous Yacc Notes

1.  By default, `yacc.py` relies on `lex.py` for tokenizing. However, an
    alternative tokenizer can be supplied as follows:

        parser = yacc.parse(lexer=x)

    in this case, `x` must be a Lexer object that minimally has a
    `x.token()` method for retrieving the next token. If an input string
    is given to `yacc.parse()`, the lexer must also have an `x.input()`
    method.

2.  To print copious amounts of debugging during parsing, use:

        parser.parse(input_text, debug=True)     

3.  Since LR parsing is driven by tables, the performance of the parser
    is largely independent of the size of the grammar. The biggest
    bottlenecks will be the lexer and the complexity of the code in your
    grammar rules.

4.  `yacc()` also allows parsers to be defined as classes and as
    closures (see the section on alternative specification of lexers).
    However, be aware that only one parser may be defined in a single
    module (source file). There are various error checks and validation
    steps that may issue confusing error messages if you try to define
    multiple parsers in the same source file.

## Multiple Parsers and Lexers

In advanced parsing applications, you may want to have multiple parsers
and lexers.

As a general rules this isn\'t a problem. However, to make it work, you
need to carefully make sure everything gets hooked up correctly. First,
make sure you save the objects returned by `lex()` and `yacc()`. For
example:

    lexer  = lex.lex()       # Return lexer object
    parser = yacc.yacc()     # Return parser object

Next, when parsing, make sure you give the `parse()` function a
reference to the lexer it should be using. For example:

    parser.parse(text,lexer=lexer)

If you forget to do this, the parser will use the last lexer
created\--which is not always what you want.

Within lexer and parser rule functions, these objects are also
available. In the lexer, the \"lexer\" attribute of a token refers to
the lexer object that triggered the rule. For example:

    def t_NUMBER(t):
       r'\d+'
       ...
       print(t.lexer)           ## Show lexer object

In the parser, the \"lexer\" and \"parser\" attributes refer to the
lexer and parser objects respectively:

    def p_expr_plus(p):
       'expr : expr PLUS expr'
       ...
       print(p.parser)          # Show parser object
       print(p.lexer)           # Show lexer object

If necessary, arbitrary attributes can be attached to the lexer or
parser object. For example, if you wanted to have different parsing
modes, you could attach a mode attribute to the parser object and look
at it later.

## Advanced Debugging

Debugging a compiler is typically not an easy task. PLY provides some
diagostic capabilities through the use of Python\'s `logging` module.
The next two sections describe this:

### Debugging the lex() and yacc() commands

Both the `lex()` and `yacc()` commands have a debugging mode that can be
enabled using the `debug` flag. For example:

    lex.lex(debug=True)
    yacc.yacc(debug=True)

Normally, the output produced by debugging is routed to either standard
error or, in the case of `yacc()`, to a file `parser.out`. This output
can be more carefully controlled by supplying a logging object. Here is
an example that adds information about where different debugging
messages are coming from:

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

If you supply a custom logger, the amount of debugging information
produced can be controlled by setting the logging level. Typically,
debugging messages are either issued at the `DEBUG`, `INFO`, or
`WARNING` levels.

PLY\'s error messages and warnings are also produced using the logging
interface. This can be controlled by passing a logging object using the
`errorlog` parameter:

    lex.lex(errorlog=log)
    yacc.yacc(errorlog=log)

If you want to completely silence warnings, you can either pass in a
logging object with an appropriate filter level or use the `NullLogger`
object defined in either `lex` or `yacc`. For example:

    yacc.yacc(errorlog=yacc.NullLogger())

### Run-time Debugging

To enable run-time debugging of a parser, use the `debug` option to
parse. This option can either be an integer (which turns debugging on or
off) or an instance of a logger object. For example:

    log = logging.getLogger()
    parser.parse(input,debug=log)

If a logging object is passed, you can use its filtering level to
control how much output gets generated. The `INFO` level is used to
produce information about rule reductions. The `DEBUG` level will show
information about the parsing stack, token shifts, and other details.
The `ERROR` level shows information related to parsing errors.

For very complicated problems, you should pass in a logging object that
redirects to a file where you can more easily inspect the output after
execution.

## Using Python -OO Mode

Because of PLY\'s reliance on docstrings, it is not compatible with
[-OO]{.title-ref} mode of the interpreter (which strips docstrings). If
you want to support this, you\'ll need to write a decorator or some
other tool to attach docstrings to functions. For example::

    def _(doc):
        def decorate(func):
            func.__doc__ = doc
            return func
        return decorate

    @_("assignment : expr PLUS expr")
    def p_assignment(p):
        ...

PLY does not provide such a decorator by default.

## Where to go from here?

The `examples` directory of the PLY distribution contains several simple
examples. Please consult a compilers textbook for the theory and
underlying implementation details or LR parsing.
