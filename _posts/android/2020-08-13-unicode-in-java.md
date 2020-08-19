---
title: Unicode是啥，在Android&Java中怎么用
tags: Android Java Unicode UTF-16
published: true
layout: article
---

首先Unicode是啥呢？Unicode是一套统一的字符表，表里共有1,114,112（0~0x10FFFF）个码位（Code Point），每一个码位都是一个独一无二的数字，它被Unicode赋予一个字符或者是Emoji等等。目前已经加入到表中的字符大约有14w个，还用充足的空间可用。码位在Unicode中的表示方式是`U+`开头，后边加上16进制表示的字符表序号，比如`U+1E00`，它的含义就是Unicode表中的第7680个字符，也就是字符Ḁ。

<!--more-->

码位的解释，摘自StackOverFlow:
> A code point is the atomic unit (irreducible unit) of information. Text is a sequence of code points. Each code point is a number which is given meaning by the Unicode standard.

针对Unicode字符集，有不同的编码方案，比如UTF-8(Unicode Transformation Format 8-bit), UTF-16, UTF-32，它们可以将Unicode中的码位，也就是一个Unicode中的唯一数字，转化成二进制的表示方式。其中UTF-8是兼容ASCII的编码方式，也是主流的编码方式。


## 专有名词解释（Terminology）
Unicode是一个字符表，它兼容了已有的编码标准。这些已有的编码标准包含了字符，以及非字符。这些非字符就包括各种`Glyphs`。所以一个字符`Character`(特指我们最终看到的)，和`Code Point`(码位)并不是一一对应的，

在继续下去之前，先介绍一些属于概念：
* Character: 字符，表示的是有实际语义的最小文字单元，也就是作为用户的我们，在屏幕上看到的一个字。
* Code Point: [上边已经介绍过了](#unicode是啥在java中怎么用)
* Glyph: 字形，属于字体的一部分，它是一张图，用来表示一个字符，或者是字符的一部分，它是字符`Character`渲染之后结果，一个字符渲染之后，并不一定只有一个字形。
* Grapheme：用户能识别出来的一个图形单元，或者用户在终端设备上所认为的字符。它由一个或者多个字形表示。后边就用它来表示我们看到的*字符*
* Code Unit: Code Point的组成单位，用来表示字符编码时的存储单位，有8位，16位，32位。

简单概括一下：在Unicode的层面，一个字符(Character)由一个Code Point表示，而在渲染之后，一个字符可能渲染成一个或多个字形(Glyph)，而我们在识别的时候，又会把一个或者多个字形(Glyph)识别成一个元素(Grapheme)，所以最终可以得出结论，**我们看到的元素（Grapheme）可能并不是只有一个Code Point表示。**

### 编码实现
根据编码方式（UTF-8，UTF-16，UTF-32）的不同，码位可能由一个或者多个Code Unit 组成。我在这里只讨论Java中对于Unicode的编码实现。

在Java中，一个Code Unit 是一个16位的值，从这里可以看出，Java中使用了UTF-16的编码实现。在Java中，一个Code Point用一个4个字节的Int类型表示，其中的低21位是有效位，高11位用0表示。在Java中，原始的Unicode表达式为`\u`后缀16进制的Unicode值。比如`\u0041`表示字符`A`，`\u00DF`表示字符`ß`。

#### Java中的char和Unicode
从`U+0000`到`U+FFFF`这个范围的字符，在Java中，刚好能用一个`char`类型表示，这个范围内的字符被称作BMP（Basic Multilingual Plane），而从`U+10000`到`U+10FFFF`范围的字符，被称作supplementary characters。所以Java中的新增字符，要怎么表示的，因为它的表示范围超出了2个字节，无法用一个`char`类型表示了。那怎么办呢？

看看Java官方的解释：

>To support supplementary characters without changing the char primitive data type and causing incompatibility with previous Java programs, supplementary characters are defined by a pair of code point values that are called surrogates. The first code point is from the high surrogates range of U+D800 to U+DBFF, and the second code point is from the low surrogates range of U+DC00 to U+DFFF. For example, the Deseret character LONG I, U+10400, is defined with this pair of surrogate values: U+D801 and U+DC00.

也就是说，超出BMP范围的字符，在Java中用一对儿Code Point来定义。这一对儿Code Point就叫做`surrogates`。其中它的第一个码位的范围是 U+D800 到 U+DBFF，第二个码位的范围是U+DC00 到 U+DFFF。

那么surrogates又是如何实现的呢？如果把一个在`U+10000`到`U+10FFFF`之间的Unicode转化成两个码位表示的Surrogates呢？

##### UTF-16
我们举例说明。以"💣"这个炸弹Emoji为例，在Java中，`"💣".length()`的结果是2，也就是说它用了两个Char来表示这个符号(Grapheme)。[查阅Unicode表](https://emojipedia.org/bomb/)，知道它的Unicode是U+1F4A3，所以它在Java中就要用到Surrogate Paier表示。那怎么表示呢？最终表示成什么呢？

这个转化的过程就是UTF-16的编码方式了。遵循以下几个步骤：
1. Code Point 减去0x010000，这一步去掉了高位，只保留了较低的16位。
   * 0x1F4A3 - 0x010000 = 0xF4A3
2. 把上一步得到的结果，取高10位，然后加上0xD800
   * 0xF4A3>>10 + 0xD800 = 0xD83D
3. 把第一步得到的结果取低10位，然后加上0xDC00
   * (0xF4A3 & 0x3FF) + 0xDC00 = 0xDCA3.

按照以上步骤，就得到了Bomb符号的Surrogate Pair的高低两部分：high surrogate是0xD83D，low surrogate是0xDCA3。 组合之后的结果表示成原始的Unicode：`\uD83D\uDCA3`。拿到Java中，打印这个Unicode，发现就是"💣"。证实结果没错。

当然，除了使用原始的Unicode表示方式，也可以使用另外一种方式：

```java
String bomb = new String(new int[] {0xD83D, 0xDCA3}, 0, 2);
/**
 * Allocates a new {@code String} that contains characters from a subarray
 * of the <a href="Character.html#unicode">Unicode code point</a> array
 * argument.  The {@code offset} argument is the index of the first code
 * point of the subarray and the {@code count} argument specifies the
 * length of the subarray.  The contents of the subarray are converted to
 * {@code char}s; subsequent modification of the {@code int} array does not
 * affect the newly created string.
 */
public String(int[] codePoints, int offset, int count) {
    //...
}
```

补充，除了Bomb之外，还有一些更特殊的emoji，比如"🤦‍♀️"这个Emoji，它的Code Points是"\uD83E\uDD26\u200D\u2640\uFE0F"。这就对应了[前面说的](#专有名词解释terminology)，有些Grapheme和Code Point 不是一一对应的，这种情况下，是多个Code Point（4个）对应渲染成了一个Grapheme，也就是最终的Emoji。

## 参考文档

* [What's the difference between a character, a code point, a glyph and a grapheme?](https://stackoverflow.com/questions/27331819/whats-the-difference-between-a-character-a-code-point-a-glyph-and-a-grapheme)
* [which-encoding-does-java-uses-utf-8-or-utf-16](https://stackoverflow.com/questions/39955169/which-encoding-does-java-uses-utf-8-or-utf-16)
* [Forms of Unicode](https://icu-project.org/docs/papers/forms_of_unicode/#h2)