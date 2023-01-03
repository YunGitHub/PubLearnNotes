---
layout: post
author: smartsi
title: ANTLR4 如何编写语法文件之语法词汇
date: 2023-01-03 15:15:01
tags:
  - Antlr

categories: Antlr
permalink: antlr-grammar-lexicon
---

ANTLR 中的词汇大多数程序员可能都熟悉，因为它遵循 C 语言及其派生语言的语法，此外还对语法进行了一些扩展。

## 1. 注释 Comments

ANTLR 支持单行、多行以及 Javadoc 风格的注释：
```
/** 这是说明三种注释的语法示例
 */
grammar T;
/*多行
* 注释
*/

/** 此规则匹配自定义语言中的一个声明 */
decl : ID ; // 匹配一个变量名
```
Javadoc 风格的注释不会被忽略，他们会被送入语法分析器，它们只能出现在语法和任意规则的开头

> Javadoc 风格的注释是 Java 代码的一种注释风格，可以用 javadoc 命令生成 API 文档。

## 2. 标识符 Identifiers

词条 Token 名称以及词法解析器规则(词法规则)名称总是以大写字母开头，其中大写字母由 Java 的 Character.isUpperCase 方法定义。语法解析器规则(语法规则)名称总是以小写字母开头（Character.isUpperCase 方法返回 false）。首字母之后的字符可以是大写字母，也可以是小写字母，也可以是数字或者下划线。如下是简单的例子:
```
// 词条或者词法解析器规则名称
ID, LPAREN, RIGHT_CURLY
// 语法解析器规则名称
expr, simpleDeclarator, d2, header_file
```

> 词法规则名总是以大写字母开头；语法规则名总是以小写字母开头

与 Java 类似，ANTLR 也允 Unicode 字符出现在标识符中：

![](https://github.com/sjf0115/ImageBucket/blob/main/Antlr/antlr-grammar-lexicon-1.png?raw=true)

为了支持 Unicode 的词法解析器规则和语法解析器规则，ANTLR 使用如下规则：
```
ID : a=NameStartChar NameChar*
     {  
     if ( Character.isUpperCase(getText().charAt(0)) ) setType(TOKEN_REF);
     else setType(RULE_REF);
     }  
   ;
```
`NameChar` 规则标识可以作为标识符的有效字符，`NameStartChar` 规则标识可以作为标识符(规则、词条或者标签名称)首字母的字符列表。这些字符大致与 Java Character 类的 isJavaIdentifierPart 和 isJavaIdentifierStart 相对应：
```
fragment
NameChar
   : NameStartChar
   | '0'..'9'
   | '_'
   | '\u00B7'
   | '\u0300'..'\u036F'
   | '\u203F'..'\u2040'
   ;
fragment
NameStartChar
   : 'A'..'Z' | 'a'..'z'
   | '\u00C0'..'\u00D6'
   | '\u00D8'..'\u00F6'
   | '\u00F8'..'\u02FF'
   | '\u0370'..'\u037D'
   | '\u037F'..'\u1FFF'
   | '\u200C'..'\u200D'
   | '\u2070'..'\u218F'
   | '\u2C00'..'\u2FEF'
   | '\u3001'..'\uD7FF'
   | '\uF900'..'\uFDCF'
   | '\uFDF0'..'\uFFFD'
   ;
```
如果语法文件不是 `UTF-8` 编码格式，请确保在 ANTLR 工具上使用 `-encoding` 选项，以便 ANTLR 正确的读取字符。

## 3. 文本常量 Literals

ANTLR 不像大多数语言那样区分字符常量和字符串常量。长度为一个或多个字符的文本字符串都使用单引号包裹，例如 `';'`，`'if'`，`'>='` 以及 `'\"`(包含单引号字符的单字符字符串)。文本常量从不包含正则表达式。

文本常量可以包含形式为`'\uXXXX'`或`'\U{XXXXXX}'`的 Unicode 转义序列，其中'XXXX'是十六进制的 Unicode 字符值。例如，`'\u00E8'` 是法语字母 `è`，而 `\u{1F4A9}` 是著名的表情符号`💩`。

ANTLR 还可以识别常见的特殊转义序列：`'\n'`(换行符)、`'\r'`(回车)、`'\t'`(制表符)、`'\b'`(退格)和`'\f'`(换行)。你可以直接使用它们或者使用它们的 Unicode 转义序列：
```
grammar Foreign;
a : '外' ;
```

ANTLR 生成的识别器假设语法中的字符都是 Unicode 字符。ANTLR 运行时库对输入文件编码做出的假设取决于目标语言。对于 Java 目标语言，运行时库假设文件编码是 UTF-8 格式。使用 CharStreams 中的工厂方法，您可以指定不同的编码格式。

## 4. 动作 Actions

动作是使用目标语言编写的代码块。您可以在语法中的不同地方使用动作，但语法格式是一样的：用花括号括起来的任意文本。如果右花括号字符在字符串或注释中，则不需要转义: `"}"` 或者 `/*}*/`。在花括号平衡的情况下，无需转义 `}`，例如 `{{...}}`。其他情况下，额外的花括号需要使用反斜杠转义：`{\{}` 或 `{\}}`。动作代码应该符合语言选项指定的目标语言的语法。

嵌入式代码可以出现在：`@header` 和 `@member` 命名的动作、词法解析器规则和语法解析器规则、异常捕获区、语法解析器规则的属性部分(返回值、参数以及局部变量)以及一些规则元素的选项中。

## 5. 关键词

如下是 ANTLR 语法中保留关键字列表：
```
import, fragment, lexer, parser, grammar, returns,
locals, throws, catch, finally, mode, options, tokens
```
另外，尽管 `rule` 不是关键字，但不要使用 `rule` 作为规则名。此外，不要使用目标语言(例如，Java)中的任何关键字作为词条 Token、标签或者规则名。例如，`if` 规则将生成一个名为 `if` 的函数。这显然不能通过编译。


> 原文：[Grammar Lexicon](https://github.com/antlr/antlr4/blob/master/doc/lexicon.md)
