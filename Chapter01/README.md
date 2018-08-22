# 万花筒：教程介绍与词法分析器
* 教程简介
* 基础语言
* Lexer
## 1.1 教程介绍

> 欢迎使用“使用LLVM实现语言”教程。本教程将介绍一种简单语言的实现，展示它的乐趣和简单性。本教程将帮助您了解并开始构建可扩展到其他语言的框架。本教程中的代码也可以用作破解其他LLVM特定事物的操场。

> 本教程的目标是逐步推出我们的语言，描述它是如何随着时间的推移而构建的。这将让我们涵盖相当广泛的语言设计和LLVM特定的使用问题，一直展示和解释它的代码，而不会让您在预先获得大量细节。

> 有必要提前指出本教程实际上是专门教授编译器技术和LLVM，而不是教授现代和理智的软件工程原理。实际上，这意味着我们将采用一些快捷方式来简化说明。例如，代码在整个地方使用全局变量，不使用访问者等漂亮的设计模式 ......但它非常简单。如果您深入挖掘并使用代码作为未来项目的基础，那么解决这些缺陷并不难。

> 我试图将这个教程放在一起，如果你已经熟悉或者对各个部分不感兴趣，那么这些章节很容易被跳过。本教程的结构是：

* 第1章：Kaleidoscope语言简介及其Lexer的定义 - 这显示了我们的目标以及我们希望它执行的基本功能。为了使本教程最容易理解和破解，我们选择用C ++实现所有内容，而不是使用词法分析器和解析器生成器。LLVM显然可以很好地使用这些工具，如果您愿意，可以随意使用它。
* 第2章：实现解析器和AST - 使用词法分析器，我们可以讨论解析技术和基本的AST构造。本教程描述了递归下降解析和运算符优先级解析。第1章或第2章中没有任何内容是特定于LLVM的，此时代码甚至不在LLVM中链接。:)
* 第3章：LLVM IR的代码生成 - 在AST就绪的情况下，我们可以展示LLVM IR的生成是多么容易。
* 第4章：添加JIT和优化器支持 - 因为很多人都有兴趣将LLVM用作JIT，我们将直接进入它并向您展示添加JIT支持所需的3行。LLVM在许多其他方面也很有用，但这是展示其强大功能的一种简单而“性感”的方式。:)
* 第5章：扩展语言：控制流 - 随着语言的启动和运行，我们将展示如何使用控制流操作扩展它（if / then / else和'for'循环）。这让我们有机会谈论简单的SSA构造和控制流程。
* 第6章：扩展语言：用户定义的操作符 - 这是一个愚蠢但有趣的章节，讨论扩展语言以让用户程序定义自己的任意一元和二元运算符（具有可赋值的优先级！）。这让我们可以构建一个重要的“语言”作为库例程。
* 第7章：扩展语言：可变变量 - 本章讨论添加用户定义的局部变量以及赋值运算符。关于这个有趣的部分是它是多么容易和琐碎构造SSA形式LLVM：没有，LLVM并没有要求您的前端构造SSA形式！
* 第8章：编译到目标文件 - 本章介绍如何获取LLVM IR并将其编译为目标文件。
* 第9章：扩展语言：调试信息 - 使用控制流，函数和可变变量构建了一个不错的编程语言，我们考虑将调试信息添加到独立可执行文件所需的内容。此调试信息将允许您在Kaleidoscope函数中设置断点，打印参数变量和调用函数 - 所有这些都来自调试器！
* 第10章：结论和其他有用的LLVM花絮 - 本章通过讨论扩展语言的可能方法来讨论本系列，还包括一系列关于“特殊主题”信息的指针，如添加垃圾收集支持，异常，调试，支持“spaghetti stacks”，以及其他一些提示和技巧。
> 在本教程结束时，我们将编写少于1000行非注释，非空白的代码行。有了这么少的代码，我们就可以为非平凡的语言构建一个非常合理的编译器，包括手写的词法分析器，解析器，AST，以及使用JIT编译器的代码生成支持。虽然其他系统可能有一些有趣的“hello world”教程，但我认为本教程的广泛性是对LLVM优势的一个很好的证明，以及为什么你应该考虑它，如果你对语言或编译器设计感兴趣。

> 关于本教程的说明：我们希望您扩展语言并自行使用它。接受代码并疯狂地破解它，编译器不需要是可怕的生物 - 使用语言可以很有趣！

## 1.2基础语言
> 本教程将使用我们称之为“ Kaleidoscope ” 的玩具语言进行说明（源自“意为美丽，形式和视图”）。Kaleidoscope是一种过程语言，允许您定义函数，使用条件，数学等。在本教程中，我们将扩展Kaleidoscope以支持if / then / else构造，for循环，用户定义的运算符，JIT使用简单的命令行界面进行编译等

> 因为我们希望保持简单，所以Kaleidoscope中唯一的数据类型是64位浮点类型（在C语言中也称为“double”）。因此，所有值都是隐式双精度，并且语言不需要类型声明。这为语言提供了非常好的简单语法。例如，以下简单示例计算 Fibonacci数：
```sh
# Compute the x'th fibonacci number.
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2)

# This expression will compute the 40th number.
fib(40)
```
> 我们还允许Kaleidoscope调用标准库函数（LLVM JIT使这完全无关紧要）。这意味着您可以在使用之前使用'extern'关键字来定义函数（这对于相互递归函数也很有用）。例如：
```sh
extern sin(arg);
extern cos(arg);
extern atan2(arg1 arg2);

atan2(sin(.4), cos(42))
```
> 第6章中包含了一个更有趣的例子，我们编写了一个小的Kaleidoscope应用程序，它以不同的放大倍数显示Mandelbrot Set。

> 让我们深入探讨这种语言的实现！

## 1.3 词法分析器
> 在实现一种语言时，首先需要的是能够处理文本文件并识别它所说的内容。传统的方法是使用“ 词法分析器 ”（又名“扫描仪”）将输入分解为“令牌”。词法分析器返回的每个标记包括标记代码和可能的一些元数据（例如，数字的数值）。首先，我们定义了可能性：
```cpp
// The lexer returns tokens [0-255] if it is an unknown character, otherwise one
// of these for known things.
enum Token {
  tok_eof = -1,

  // commands
  tok_def = -2,
  tok_extern = -3,

  // primary
  tok_identifier = -4,
  tok_number = -5,
};

static std::string IdentifierStr; // Filled in if tok_identifier
static double NumVal;             // Filled in if tok_number
```
> 我们的词法分析器返回的每个标记将是Token枚举值之一，或者它将是一个'未知'字符，如'+'，它将作为ASCII值返回。如果当前标记是标识符，则 IdentifierStr全局变量保存标识符的名称。如果当前标记是数字文字（如1.0），则NumVal保持其值。请注意，为简单起见，我们使用全局变量，这不是真正的语言实现的最佳选择:)。

> 词法分析器的实际实现是一个名为的单个函数 gettok。gettok调用该函数以从标准输入返回下一个标记。其定义开始于：
```cpp
/// gettok - Return the next token from standard input.
static int gettok() {
  static int LastChar = ' ';

  // Skip any whitespace.
  while (isspace(LastChar))
    LastChar = getchar();
```
> gettok通过调用C getchar()函数从标准输入一次读取一个字符。它在识别它们时会吃它们，并在LastChar中存储读取但未处理的最后一个字符。它要做的第一件事就是忽略令牌之间的空格。这是通过上面的循环完成的。

> 接下来gettok需要做的是识别标识符和特定关键字，如“def”。Kaleidoscope通过这个简单的循环实现了这一点：
```cpp
if (isalpha(LastChar)) { // identifier: [a-zA-Z][a-zA-Z0-9]*
  IdentifierStr = LastChar;
  while (isalnum((LastChar = getchar())))
    IdentifierStr += LastChar;

  if (IdentifierStr == "def")
    return tok_def;
  if (IdentifierStr == "extern")
    return tok_extern;
  return tok_identifier;
}
```
> 请注意，此代码在设置IdentifierStr标识符时设置“全局”。此外，由于语言关键字由相同的循环匹配，我们在这里处理它们。数值类似：
```cpp
if (isdigit(LastChar) || LastChar == '.') {   // Number: [0-9.]+
  std::string NumStr;
  do {
    NumStr += LastChar;
    LastChar = getchar();
  } while (isdigit(LastChar) || LastChar == '.');

  NumVal = strtod(NumStr.c_str(), 0);
  return tok_number;
}
```
> 这是处理输入的非常简单的代码。从输入读取数值时，我们使用C strtod函数将其转换为我们存储的数值NumVal。请注意，这没有进行足够的错误检查：它将错误地读取“1.23.45.67”并像处理“1.23”一样处理它。随意扩展它:)。接下来我们处理评论：
```cpp
if (LastChar == '#') {
  // Comment until end of line.
  do
    LastChar = getchar();
  while (LastChar != EOF && LastChar != '\n' && LastChar != '\r');

  if (LastChar != EOF)
    return gettok();
}
```
> 我们通过跳到行尾来处理注释，然后返回下一个令牌。最后，如果输入与上述情况之一不匹配，则它是像“+”这样的运算符字符或文件的结尾。这些代码使用以下代码处理：
```cpp
  // Check for end of file.  Don't eat the EOF.
  if (LastChar == EOF)
    return tok_eof;

  // Otherwise, just return the character as its ascii value.
  int ThisChar = LastChar;
  LastChar = getchar();
  return ThisChar;
}
```
有了这个，我们有了基本的万花筒语言的完整词法分析器（Lexer 的完整代码清单可以在本教程的下一章中找到）。接下来，我们将构建一个简单的解析器，使用它来构建抽象语法树。当我们有这个时，我们将包含一个驱动程序，以便您可以一起使用词法分析器和解析器。

[下一页：实现解析器和AST](../Chapter02/README.md)