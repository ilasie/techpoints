+++
date = '2026-08-22T15:30:59+08:00'
draft = false
title = 'Vim/Neovim中的modeline'
+++

modeline（译为**模式行**）是Vim/Neovim原生支持的一种控制手段。使用modeline可以将配置语句嵌入到特定文件中，以形成局部配置。modeline选项是默认开启的。当它开启时，Vim/Neovim将会在加载缓冲区时扫描文件的首尾数行以查找modeline内容。可以用`modelines`控制首尾的扫描行数，`modelines`的默认值为5。

modeline有两种写法。使用
```vim
:help modeline
```
可以获得详细的介绍，这里给出简单版本：\
modeline一般以注释形式嵌入文本。以C为例，下面的写法是合法的。以下两种注释的差异只是为了说明，任何形式的注释行都可以作为modeline。\
第一种写法较为简化: 
```c
// vi:ai:sw=8
// vim: ai sw=8
```
此写法可以使用`vi:`、`vim:`和`ex:`开头。在此写法中，可以使用空格或`:`分割选项。每个选项都应该是`:set`命令的一个可接受参数。\
第二种写法是标准的，特别地，它可以在某些版本的Vi上工作：
```c
/* vim: set ai sw=8 :*/
/* Vim: set ai sw=8: */
```
此写法可以使用`vi:`、`vim:`、`Vim:`和`ex:`开头。在此写法中，**必须**用空格分割选项，且注意标准形式的modeline必须以`:`结尾。选项和最后的`:`之间可以有空格。

modeline实际上可以执行任何Vimscript，为了防止恶意代码注入，Vim/Neovim默认启用`nomodelineexpr`以禁用modeline中的表达式。可以使用
```vim
:set modelineexpr
```
开启modeline的表达式执行功能。
