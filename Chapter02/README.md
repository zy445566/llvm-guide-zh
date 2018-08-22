# 2.万花筒：实现解析器和
* 第2章简介
* 抽象语法树（AST）
* 分析基础知识
* 基本表达解析
* 二进制表达式解析
* 解析其余部分
* 驱动程序
* 结论
* 完整的代码清单
## 2.1 第2章介绍
> 欢迎阅读“ 使用LLVM实现语言 ”教程的第2章。本章介绍如何使用第1章中构建的词法分析器为我们的Kaleidoscope语言构建完整的 解析器。一旦我们有了解析器，我们将定义并构建一个抽象语法树（AST）。

> 我们将构建的解析器使用递归下降解析和 操作符优先解析的组合来解析Kaleidoscope语言（后者用于二进制表达式，前者用于其他所有语言）。在我们解析之前，让我们来谈谈解析器的输出：抽象语法树。

# 2.2 抽象语法树（AST）
> 程序的AST以这样的方式捕获其行为，即编译器的后续阶段（例如代码生成）很容易解释。我们基本上希望语言中每个构造都有一个对象，AST应该对语言进行密切建模。在Kaleidoscope中，我们有表达式，原型和函数对象。我们先从表达式开始：
```cpp
/// ExprAST - Base class for all expression nodes.
class ExprAST {
public:
  virtual ~ExprAST() {}
};

/// NumberExprAST - Expression class for numeric literals like "1.0".
class NumberExprAST : public ExprAST {
  double Val;

public:
  NumberExprAST(double Val) : Val(Val) {}
};
```
> 上面的代码显示了基本ExprAST类的定义和我们用于数字文字的一个子类。关于此代码的重要注意事项是NumberExprAST类将文字的数值捕获为实例变量。这允许编译器的后续阶段知道存储的数值是什么。

> 现在我们只创建AST，因此它们没有有用的访问器方法。例如，添加虚拟方法来非常容易地打印代码是非常容易的。以下是我们将在Kaleidoscope语言的基本形式中使用的其他表达式AST节点定义：
```cpp
/// VariableExprAST - Expression class for referencing a variable, like "a".
class VariableExprAST : public ExprAST {
  std::string Name;

public:
  VariableExprAST(const std::string &Name) : Name(Name) {}
};

/// BinaryExprAST - Expression class for a binary operator.
class BinaryExprAST : public ExprAST {
  char Op;
  std::unique_ptr<ExprAST> LHS, RHS;

public:
  BinaryExprAST(char op, std::unique_ptr<ExprAST> LHS,
                std::unique_ptr<ExprAST> RHS)
    : Op(op), LHS(std::move(LHS)), RHS(std::move(RHS)) {}
};

/// CallExprAST - Expression class for function calls.
class CallExprAST : public ExprAST {
  std::string Callee;
  std::vector<std::unique_ptr<ExprAST>> Args;

public:
  CallExprAST(const std::string &Callee,
              std::vector<std::unique_ptr<ExprAST>> Args)
    : Callee(Callee), Args(std::move(Args)) {}
};
```
> 这都是（有意）相当直接：变量捕获变量名称，二元运算符捕获它们的操作码（例如'+'），并且调用捕获函数名称以及任何参数表达式的列表。关于我们的AST的一个好处是它捕获语言功能而不谈论语言的语法。请注意，没有讨论二元运算符的优先级，词法结构等。

> 对于我们的基本语言，这些是我们将定义的所有表达式节点。因为它没有条件控制流程，所以它不是图灵完备的; 我们将在以后的文章中解决这个问题。接下来我们需要的两件事是讨论函数接口的方法，以及讨论函数本身的方法：
```cpp
/// PrototypeAST - This class represents the "prototype" for a function,
/// which captures its name, and its argument names (thus implicitly the number
/// of arguments the function takes).
class PrototypeAST {
  std::string Name;
  std::vector<std::string> Args;

public:
  PrototypeAST(const std::string &name, std::vector<std::string> Args)
    : Name(name), Args(std::move(Args)) {}

  const std::string &getName() const { return Name; }
};

/// FunctionAST - This class represents a function definition itself.
class FunctionAST {
  std::unique_ptr<PrototypeAST> Proto;
  std::unique_ptr<ExprAST> Body;

public:
  FunctionAST(std::unique_ptr<PrototypeAST> Proto,
              std::unique_ptr<ExprAST> Body)
    : Proto(std::move(Proto)), Body(std::move(Body)) {}
};
```
> 在Kaleidoscope中，只使用参数计数来输入函数。由于所有值都是双精度浮点数，因此每个参数的类型不需要存储在任何位置。在更具侵略性和现实性的语言中，“ExprAST”类可能具有类型字段。

> 有了这个脚手架，我们现在可以讨论在Kaleidoscope中解析表达式和函数体。

## 2.3 解析基础
> 现在我们要构建一个AST，我们需要定义解析器代码来构建它。这里的想法是我们要解析类似“x + y”（由词法分析器返回三个标记）到AST中，这可以通过这样的调用生成：
```cpp
auto LHS = llvm::make_unique<VariableExprAST>("x");
auto RHS = llvm::make_unique<VariableExprAST>("y");
auto Result = std::make_unique<BinaryExprAST>('+', std::move(LHS),
                                              std::move(RHS));
```
> 为此，我们首先定义一些基本的帮助程序：
```cpp
/// CurTok/getNextToken - Provide a simple token buffer.  CurTok is the current
/// token the parser is looking at.  getNextToken reads another token from the
/// lexer and updates CurTok with its results.
static int CurTok;
static int getNextToken() {
  return CurTok = gettok();
}
```
> 这在词法分析器周围实现了一个简单的标记缓冲区。这允许我们在词法分析器返回时提前查看一个标记。我们的解析器中的每个函数都假定CurTok是需要解析的当前标记。
```cpp
/// LogError* - These are little helper functions for error handling.
std::unique_ptr<ExprAST> LogError(const char *Str) {
  fprintf(stderr, "LogError: %s\n", Str);
  return nullptr;
}
std::unique_ptr<PrototypeAST> LogErrorP(const char *Str) {
  LogError(Str);
  return nullptr;
}
```
> 该LogError程序是我们的解析器将用来处理错误的简单的辅助程序。我们的解析器中的错误恢复不是最好的，并且不是特别用户友好的，但它对我们的教程来说已经足够了。这些例程使得在具有各种返回类型的例程中更容易处理错误：它们总是返回null。

> 有了这些基本的辅助函数，我们就可以实现我们语法的第一部分：数字文字。

## 2.4 基本表达式解析
> 我们从数字文字开始，因为它们是最简单的处理方式。对于语法中的每个作品，我们将定义一个解析该作品的函数。对于数字文字，我们有：
```cpp
/// numberexpr ::= number
static std::unique_ptr<ExprAST> ParseNumberExpr() {
  auto Result = llvm::make_unique<NumberExprAST>(NumVal);
  getNextToken(); // consume the number
  return std::move(Result);
}
```
> 这个例程非常简单：它希望在当前令牌是tok_number令牌时被调用。它获取当前数字值，创建NumberExprAST节点，将词法分析器前进到下一个标记，最后返回。

> 这有一些有趣的方面。最重要的一点是，这个例程会占用与生产相对应的所有令牌，并返回带有下一个令牌（它不是语法生成的一部分）的词法分析器缓冲区。这是递归下降解析器的一种相当标准的方法。有关更好的示例，括号运算符的定义如下：
```cpp
/// parenexpr ::= '(' expression ')'
static std::unique_ptr<ExprAST> ParseParenExpr() {
  getNextToken(); // eat (.
  auto V = ParseExpression();
  if (!V)
    return nullptr;

  if (CurTok != ')')
    return LogError("expected ')'");
  getNextToken(); // eat ).
  return V;
}
```
> 这个函数说明了解析器的一些有趣的东西：

>  1）它显示了我们如何使用LogError例程。调用时，此函数需要当前标记为'（'标记，但在解析子表达式后，可能没有'）'等待。例如，如果用户输入“（4 x”而不是“（4）”，解析器应该发出错误。因为错误可能发生，解析器需要一种方式来指示它们发生了：在我们的解析器中，我们返回null错误。

>  2）这个函数的另一个有趣的方面是它通过调用使用递归ParseExpression（我们很快就会看到它 ParseExpression可以调用ParseParenExpr）。这很强大，因为它允许我们处理递归语法，并使每个生产变得非常简单。请注意，括号不会导致AST节点本身的构造。虽然我们可以这样做，但括号中最重要的作用是引导解析器并提供分组。解析器构造AST后，不需要括号。

> 下一个简单的生产是用于处理变量引用和函数调用：
```cpp
/// identifierexpr
///   ::= identifier
///   ::= identifier '(' expression* ')'
static std::unique_ptr<ExprAST> ParseIdentifierExpr() {
  std::string IdName = IdentifierStr;

  getNextToken();  // eat identifier.

  if (CurTok != '(') // Simple variable ref.
    return llvm::make_unique<VariableExprAST>(IdName);

  // Call.
  getNextToken();  // eat (
  std::vector<std::unique_ptr<ExprAST>> Args;
  if (CurTok != ')') {
    while (1) {
      if (auto Arg = ParseExpression())
        Args.push_back(std::move(Arg));
      else
        return nullptr;

      if (CurTok == ')')
        break;

      if (CurTok != ',')
        return LogError("Expected ')' or ',' in argument list");
      getNextToken();
    }
  }

  // Eat the ')'.
  getNextToken();

  return llvm::make_unique<CallExprAST>(IdName, std::move(Args));
}
```
> 此例程遵循与其他例程相同的样式。（如果当前令牌是tok_identifier令牌，则期望被调用）。它还有递归和错误处理。这方面的一个令人感兴趣的方面是，它使用先行，以确定是否在当前标识符是一个独立变量的参考，或者如果它是一个函数调用的表达。它通过检查标识符后面的标记是否是'（'标记，根据需要构造一个VariableExprAST或 一个CallExprAST节点来处理这个问题。

> 现在我们已经拥有了所有简单的表达式解析逻辑，我们可以定义一个辅助函数将它们组合成一个入口点。我们称这类表达式为“主要”表达式，原因将在本教程后面更加清晰。为了解析任意的主表达式，我们需要确定它是什么类型的表达式：
```cpp
/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
static std::unique_ptr<ExprAST> ParsePrimary() {
  switch (CurTok) {
  default:
    return LogError("unknown token when expecting an expression");
  case tok_identifier:
    return ParseIdentifierExpr();
  case tok_number:
    return ParseNumberExpr();
  case '(':
    return ParseParenExpr();
  }
}
```
> 现在你看到了这个函数的定义，为什么我们可以在各种函数中假设CurTok的状态更为明显。这使用预测来确定正在检查哪种表达式，然后使用函数调用对其进行解析。

> 现在处理了基本表达式，我们需要处理二进制表达式。它们有点复杂。

## 2.5 二进制表达式解析
> 二进制表达式很难解析，因为它们通常是模糊的。例如，当给定字符串“x + y * z”时，解析器可以选择将其解析为“（x + y）* z”或“x +（y * z）”。对于数学中的常见定义，我们期望后面的解析，因为“*”（乘法）具有比“+”（加法）更高的优先级。

> 有很多方法可以解决这个问题，但优雅而有效的方法是使用Operator-Precedence Parsing。此解析技术使用二元运算符的优先级来指导递归。首先，我们需要一个优先级表：
```cpp
/// BinopPrecedence - This holds the precedence for each binary operator that is
/// defined.
static std::map<char, int> BinopPrecedence;

/// GetTokPrecedence - Get the precedence of the pending binary operator token.
static int GetTokPrecedence() {
  if (!isascii(CurTok))
    return -1;

  // Make sure it's a declared binop.
  int TokPrec = BinopPrecedence[CurTok];
  if (TokPrec <= 0) return -1;
  return TokPrec;
}

int main() {
  // Install standard binary operators.
  // 1 is lowest precedence.
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
  BinopPrecedence['*'] = 40;  // highest.
  ...
}
```
> 对于万花筒的基本形式，我们只支持4个二元运算符（这显然可以由你，我们的勇敢和无畏的读者扩展）。该GetTokPrecedence函数返回当前标记的优先级，如果标记不是二元运算符，则返回-1。使用地图可以轻松添加新运算符，并清楚地表明算法不依赖于所涉及的特定运算符，但是很容易消除映射并在GetTokPrecedence函数中进行比较 。（或者只使用固定大小的数组）。

> 通过上面定义的帮助程序，我们现在可以开始解析二进制表达式。运算符优先级解析的基本思想是将具有可能不明确的二元运算符的表达式分解为多个部分。例如，考虑表达式“a + b +（c + d）* e * f + g”。运算符优先级解析将此视为由二元运算符分隔的主表达式流。因此，它将首先解析主要的主要表达式“a”，然后它将看到对[+，b] [+，（c + d）] [*，e] [*，f]和[+，g] ]。请注意，因为括号是主表达式，所以二进制表达式解析器根本不需要担心嵌套的子表达式，如（c + d）。

> 首先，表达式是一个主表达式，可能后跟一系列[binop，primaryexpr]对：
```cpp
/// expression
///   ::= primary binoprhs
///
static std::unique_ptr<ExprAST> ParseExpression() {
  auto LHS = ParsePrimary();
  if (!LHS)
    return nullptr;

  return ParseBinOpRHS(0, std::move(LHS));
}
```
> ParseBinOpRHS是为我们解析对序列的函数。它需要一个优先级和一个指向目前已解析的部分的表达式的指针。请注意，“x”是一个完全有效的表达式：因此，“binoprhs”被允许为空，在这种情况下，它返回传递给它的表达式。在上面的示例中，代码将“a”的表达式传递给，ParseBinOpRHS并且当前标记为“+”。

> 传入的优先级值ParseBinOpRHS表示允许该函数吃的 最小运算符优先级。例如，如果当前对流是[+，x]并且ParseBinOpRHS以40的优先级传递，则它不会消耗任何标记（因为'+'的优先级仅为20）。考虑到这一点，ParseBinOpRHS 从以下开始：
```cpp
/// binoprhs
///   ::= ('+' primary)*
static std::unique_ptr<ExprAST> ParseBinOpRHS(int ExprPrec,
                                              std::unique_ptr<ExprAST> LHS) {
  // If this is a binop, find its precedence.
  while (1) {
    int TokPrec = GetTokPrecedence();

    // If this is a binop that binds at least as tightly as the current binop,
    // consume it, otherwise we are done.
    if (TokPrec < ExprPrec)
      return LHS;
```
> 此代码获取当前令牌的优先级，并检查是否过低。因为我们将无效标记定义为优先级为-1，所以此检查隐式地知道当标记流用完二元运算符时，对流结束。如果此检查成功，我们知道该令牌是二元运算符，并且它将包含在此表达式中：
```cpp
// Okay, we know this is a binop.
int BinOp = CurTok;
getNextToken();  // eat binop

// Parse the primary expression after the binary operator.
auto RHS = ParsePrimary();
if (!RHS)
  return nullptr;
```
> 因此，此代码吃掉（并记住）二元运算符，然后解析后面的主表达式。这构建了整个对，第一个是运行示例的[+，b]。

> 现在我们解析了表达式的左侧和一对RHS序列，我们必须决定表达式关联的方式。特别是，我们可以有“（a + b）binop unarsed”或“a +（b binop unparsed）”。为了确定这一点，我们展望“binop”以确定其优先级，并将其与BinOp的优先级（在这种情况下为“+”）进行比较：
```cpp
// If BinOp binds less tightly with RHS than the operator after RHS, let
// the pending operator take RHS as its LHS.
int NextPrec = GetTokPrecedence();
if (TokPrec < NextPrec) {
```
如果binop在“RHS”右边​​的优先级低于或等于我们当前运算符的优先级，那么我们知道括号关联为“（a + b）binop ...”。在我们的示例中，当前运算符为“+”，下一个运算符为“+”，我们知道它们具有相同的优先级。在这种情况下，我们将为“a + b”创建AST节点，然后继续解析：
```cpp
      ... if body omitted ...
    }

    // Merge LHS/RHS.
    LHS = llvm::make_unique<BinaryExprAST>(BinOp, std::move(LHS),
                                           std::move(RHS));
  }  // loop around to the top of the while loop.
}
```
> 在上面的例子中，这将把“a + b +”变成“（a + b）”并执行循环的下一次迭代，其中“+”作为当前标记。上面的代码将吃掉，记住并解析“（c + d）”作为主表达式，这使得当前对等于[+，（c + d）]。然后它将使用“*”作为主要右侧的binop来评估上面的“if”条件。在这种情况下，“*”的优先级高于“+”的优先级，因此将输入if条件。

> 这里留下的关键问题是“if条件如何完全解析右手边”？特别是，要为我们的示例正确构建AST，它需要将所有“（c + d）* e * f”作为RHS表达式变量。执行此操作的代码非常简单（上面两个块的代码重复上下文）：
```cpp
    // If BinOp binds less tightly with RHS than the operator after RHS, let
    // the pending operator take RHS as its LHS.
    int NextPrec = GetTokPrecedence();
    if (TokPrec < NextPrec) {
      RHS = ParseBinOpRHS(TokPrec+1, std::move(RHS));
      if (!RHS)
        return nullptr;
    }
    // Merge LHS/RHS.
    LHS = llvm::make_unique<BinaryExprAST>(BinOp, std::move(LHS),
                                           std::move(RHS));
  }  // loop around to the top of the while loop.
}
```
> 此时，我们知道我们的主要RHS的二元运算符优先于我们当前正在解析的binop。因此，我们知道任何运算符都优先于“+”的对的序列应该被一起解析并返回为“RHS”。为此，我们以递归方式调用ParseBinOpRHS指定“TokPrec + 1” 的函数作为其继续所需的最小优先级。在上面的例子中，这将导致它将“（c + d）* e * f”的AST节点作为RHS返回，然后将其设置为“+”表达式的RHS。

> 最后，在while循环的下一次迭代中，解析“+ g”片段并将其添加到AST。使用这一小段代码（14个非平凡的行），我们以非常优雅的方式正确处理完全通用的二进制表达式解析。这是对这段代码的旋风之旅，它有点微妙。我建议通过几个棘手的例子来看看它是如何工作的。

> 这包含了表达式的处理。此时，我们可以将解析器指向任意标记流并从中构建表达式，停止在不属于表达式的第一个标记处。接下来我们需要处理函数定义等。

## 2.6 解析其余的
> 缺少的是缺少功能原型的处理。在Kaleidoscope中，这些用于'extern'函数声明以及函数体定义。执行此操作的代码是直截了当的，并且不是很有趣（一旦表达式幸存下来）：
```cpp
/// prototype
///   ::= id '(' id* ')'
static std::unique_ptr<PrototypeAST> ParsePrototype() {
  if (CurTok != tok_identifier)
    return LogErrorP("Expected function name in prototype");

  std::string FnName = IdentifierStr;
  getNextToken();

  if (CurTok != '(')
    return LogErrorP("Expected '(' in prototype");

  // Read the list of argument names.
  std::vector<std::string> ArgNames;
  while (getNextToken() == tok_identifier)
    ArgNames.push_back(IdentifierStr);
  if (CurTok != ')')
    return LogErrorP("Expected ')' in prototype");

  // success.
  getNextToken();  // eat ')'.

  return llvm::make_unique<PrototypeAST>(FnName, std::move(ArgNames));
}
```
> 鉴于此，函数定义非常简单，只是一个原型加上一个表达式来实现正文：
```cpp
/// definition ::= 'def' prototype expression
static std::unique_ptr<FunctionAST> ParseDefinition() {
  getNextToken();  // eat def.
  auto Proto = ParsePrototype();
  if (!Proto) return nullptr;

  if (auto E = ParseExpression())
    return llvm::make_unique<FunctionAST>(std::move(Proto), std::move(E));
  return nullptr;
}
```
> 另外，我们支持'extern'来声明'sin'和'cos'之类的函数，以及支持用户函数的前向声明。这些'extern'只是没有身体的原型：
```cpp
/// external ::= 'extern' prototype
static std::unique_ptr<PrototypeAST> ParseExtern() {
  getNextToken();  // eat extern.
  return ParsePrototype();
}
```
> 最后，我们还让用户输入任意顶级表达式并动态评估它们。我们将通过为它们定义匿名的nullary（零参数）函数来处理这个问题：
```cpp
/// toplevelexpr ::= expression
static std::unique_ptr<FunctionAST> ParseTopLevelExpr() {
  if (auto E = ParseExpression()) {
    // Make an anonymous proto.
    auto Proto = llvm::make_unique<PrototypeAST>("", std::vector<std::string>());
    return llvm::make_unique<FunctionAST>(std::move(Proto), std::move(E));
  }
  return nullptr;
}
```
> 现在我们已经完成了所有部分，让我们构建一个小驱动程序，让我们实际执行我们构建的代码！

## 2.7。驱动程序
> 这个驱动程序只是通过顶级调度循环调用所有解析部分。这里没什么有趣的，所以我只包括顶级循环。请参阅下面的“顶级解析”部分中的完整代码。
```cpp
/// top ::= definition | external | expression | ';'
static void MainLoop() {
  while (1) {
    fprintf(stderr, "ready> ");
    switch (CurTok) {
    case tok_eof:
      return;
    case ';': // ignore top-level semicolons.
      getNextToken();
      break;
    case tok_def:
      HandleDefinition();
      break;
    case tok_extern:
      HandleExtern();
      break;
    default:
      HandleTopLevelExpression();
      break;
    }
  }
}
```
> 最有趣的部分是我们忽略顶级分号。你问，这是为什么？基本原因是，如果在命令行中键入“4 + 5”，则解析器不知道这是否是您要键入的结尾。例如，在下一行，您可以键入“def foo ...”，在这种情况下，4 + 5是顶级表达式的结尾。或者，您可以键入“* 6”，这将继续表达式。使用顶级分号允许您键入“4 + 5;”，解析器将知道您已完成。

## 2.8 结论
> 只有不到400行的注释代码（240行非注释，非空代码），我们完全定义了我们的最小语言，包括词法分析器，解析器和AST构建器。完成此操作后，可执行文件将验证Kaleidoscope代码并告诉我们它是否在语法上无效。例如，这是一个示例交互：
```sh 
$ ./a.out
ready> def foo(x y) x+foo(y, 4.0);
Parsed a function definition.
ready> def foo(x y) x+y y;
Parsed a function definition.
Parsed a top-level expr
ready> def foo(x y) x+y );
Parsed a function definition.
Error: unknown token when expecting an expression
ready> extern sin(a);
ready> Parsed an extern
ready> ^D
$
```
> 这里有很大的扩展空间。您可以定义新的AST节点，以多种方式扩展语言等。在下一部分中，我们将介绍如何从AST生成LLVM中间表示（IR）。

### 2.9 完整的代码清单
以下是我们正在运行的示例的完整代码清单。因为它使用LLVM库，我们需要将它们链接起来。为此，我们使用 llvm-config工具通知makefile /命令行有关使用哪些选项：
```sh
# Compile
clang++ -g -O3 toy.cpp `llvm-config --cxxflags`
# Run
./a.out
```
> 这是代码：[toy.cpp](./toy.cpp)

[下一步：实现代码生成到LLVM IR](../Chapter03/README.md)