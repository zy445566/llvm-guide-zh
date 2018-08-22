# 6.万花筒：扩展语言：用户定义的运算符
* 第6章简介
* 用户定义的运算符：理念
* 用户定义的二元运算符
* 用户定义的一元运算符
* 造轮子
* 完整的代码清单
## 6.1 第6章介绍
> 欢迎阅读“ 使用LLVM实现语言 ”教程的第6章。在我们的教程中，我们现在拥有一个功能完备的语言，它非常简单，但也很有用。然而，它仍然存在一个大问题。我们的语言没有很多有用的运算符（比如除法，逻辑否定，甚至除了小于之外的任何比较）。

> 本教程的这一章非常简单地将用户定义的运算符添加到简单而美观的Kaleidoscope语言中。这种离题现在在某些方面给我们一种简单而丑陋的语言，同时也是一种强大的语言。创建自己的语言的一个好处是你可以决定什么是好的或坏的。在本教程中，我们假设可以使用它作为一种显示一些有趣的解析技术的方法。

> 在本教程结束时，我们将运行一个示例呈现Mandelbrot集的 Kaleidoscope应用程序。这给出了一个使用Kaleidoscope及其功能集构建的示例。

## 6.2 用户定义的运算符：理念
> 我们将添加到Kaleidoscope的“运算符重载”比C ++等语言更通用。在C ++中，您只能重新定义现有运算符：您无法以编程方式更改语法，引入新运算符，更改优先级等。在本章中，我们将此功能添加到Kaleidoscope中，这将使用户完善支持的运算符集。

> 在这样的教程中进入用户定义的运算符的目的是展示使用手写解析器的强大功能和灵活性。到目前为止，我们实现的解析器使用递归下降语法的大多数部分和运算符优先级解析表达式。详细信息请参见第2章。通过使用运算符优先级解析，很容易让程序员在语法中引入新的运算符：语法在JIT运行时可以动态扩展。

> 我们将添加的两个特定功能是可编程的一元运算符（现在，Kaleidoscope完全没有一元运算符）以及二元运算符。一个例子是：
```sh
# Logical unary not.
def unary!(v)
  if v then
    0
  else
    1;

# Define > with the same precedence as <.
def binary> 10 (LHS RHS)
  RHS < LHS;

# Binary "logical or", (note that it does not "short circuit")
def binary| 5 (LHS RHS)
  if LHS then
    1
  else if RHS then
    1
  else
    0;

# Define = with slightly lower precedence than relationals.
def binary= 9 (LHS RHS)
  !(LHS < RHS | LHS > RHS);
```
> 许多语言都希望能够在语言本身中实现其标准运行时库。在Kaleidoscope中，我们可以在库中实现该语言的重要部分！

> 我们将这些功能的实现分解为两部分：实现对用户定义的二元运算符的支持以及添加一元运算符。

## 6.3用户定义的二元运算符
> 使用我们当前的框架，添加对用户定义的二元运算符的支持非常简单。我们首先添加对一元/二元关键字的支持：
```cpp
enum Token {
  ...
  // operators
  tok_binary = -11,
  tok_unary = -12
};
...
static int gettok() {
...
    if (IdentifierStr == "for")
      return tok_for;
    if (IdentifierStr == "in")
      return tok_in;
    if (IdentifierStr == "binary")
      return tok_binary;
    if (IdentifierStr == "unary")
      return tok_unary;
    return tok_identifier;
```
> 这只是增加了对一元和二元关键字的词法分析器支持，就像我们在前几章中所做的那样。关于我们当前AST的一个好处是，我们通过使用它们的ASCII代码作为操作码来表示具有完全泛化的二元运算符。对于我们的扩展运算符，我们将使用相同的表示，因此我们不需要任何新的AST或解析器支持。

> 另一方面，我们必须能够在“def binary |”中表示这些新运算符的定义 5“功能定义的一部分。到目前为止，在我们的语法中，函数定义的“名称”被解析为“原型”生成并进入 PrototypeASTAST节点。为了将我们新的用户定义的运算符表示为原型，我们必须PrototypeAST像这样扩展AST节点：
```cpp
/// PrototypeAST - This class represents the "prototype" for a function,
/// which captures its argument names as well as if it is an operator.
class PrototypeAST {
  std::string Name;
  std::vector<std::string> Args;
  bool IsOperator;
  unsigned Precedence;  // Precedence if a binary op.

public:
  PrototypeAST(const std::string &name, std::vector<std::string> Args,
               bool IsOperator = false, unsigned Prec = 0)
  : Name(name), Args(std::move(Args)), IsOperator(IsOperator),
    Precedence(Prec) {}

  Function *codegen();
  const std::string &getName() const { return Name; }

  bool isUnaryOp() const { return IsOperator && Args.size() == 1; }
  bool isBinaryOp() const { return IsOperator && Args.size() == 2; }

  char getOperatorName() const {
    assert(isUnaryOp() || isBinaryOp());
    return Name[Name.size() - 1];
  }

  unsigned getBinaryPrecedence() const { return Precedence; }
};
```
> 基本上，除了知道原型的名称之外，我们现在还要跟踪它是否是运算符，如果是，运算符的优先级是什么。优先级仅用于二元运算符（如下所示，它不适用于一元运算符）。既然我们有办法为用户定义的运算符表示原型，我们需要解析它：
```cpp
/// prototype
///   ::= id '(' id* ')'
///   ::= binary LETTER number? (id, id)
static std::unique_ptr<PrototypeAST> ParsePrototype() {
  std::string FnName;

  unsigned Kind = 0;  // 0 = identifier, 1 = unary, 2 = binary.
  unsigned BinaryPrecedence = 30;

  switch (CurTok) {
  default:
    return LogErrorP("Expected function name in prototype");
  case tok_identifier:
    FnName = IdentifierStr;
    Kind = 0;
    getNextToken();
    break;
  case tok_binary:
    getNextToken();
    if (!isascii(CurTok))
      return LogErrorP("Expected binary operator");
    FnName = "binary";
    FnName += (char)CurTok;
    Kind = 2;
    getNextToken();

    // Read the precedence if present.
    if (CurTok == tok_number) {
      if (NumVal < 1 || NumVal > 100)
        return LogErrorP("Invalid precedence: must be 1..100");
      BinaryPrecedence = (unsigned)NumVal;
      getNextToken();
    }
    break;
  }

  if (CurTok != '(')
    return LogErrorP("Expected '(' in prototype");

  std::vector<std::string> ArgNames;
  while (getNextToken() == tok_identifier)
    ArgNames.push_back(IdentifierStr);
  if (CurTok != ')')
    return LogErrorP("Expected ')' in prototype");

  // success.
  getNextToken();  // eat ')'.

  // Verify right number of names for operator.
  if (Kind && ArgNames.size() != Kind)
    return LogErrorP("Invalid number of operands for operator");

  return llvm::make_unique<PrototypeAST>(FnName, std::move(ArgNames), Kind != 0,
                                         BinaryPrecedence);
}
```
> 这都是相当简单的解析代码，我们在过去已经看过很多类似的代码。关于上面代码的一个有趣的部分是FnName为二元运算符设置的几行。这为新定义的“@”运算符构建了类似“binary @”的名称。然后，它利用了LLVM符号表中的符号名称允许包含任何字符的事实，包括嵌入的nul字符。

> 下一个有趣的事情是codegen支持这些二元运算符。鉴于我们当前的结构，这是我们现有二元运算符节点的默认情况的简单添加：
```cpp
Value *BinaryExprAST::codegen() {
  Value *L = LHS->codegen();
  Value *R = RHS->codegen();
  if (!L || !R)
    return nullptr;

  switch (Op) {
  case '+':
    return Builder.CreateFAdd(L, R, "addtmp");
  case '-':
    return Builder.CreateFSub(L, R, "subtmp");
  case '*':
    return Builder.CreateFMul(L, R, "multmp");
  case '<':
    L = Builder.CreateFCmpULT(L, R, "cmptmp");
    // Convert bool 0/1 to double 0.0 or 1.0
    return Builder.CreateUIToFP(L, Type::getDoubleTy(TheContext),
                                "booltmp");
  default:
    break;
  }

  // If it wasn't a builtin binary operator, it must be a user defined one. Emit
  // a call to it.
  Function *F = getFunction(std::string("binary") + Op);
  assert(F && "binary operator not found!");

  Value *Ops[2] = { L, R };
  return Builder.CreateCall(F, Ops, "binop");
}
```
> 如您所见，新代码实际上非常简单。它只是在符号表中查找适当的运算符并生成对它的函数调用。由于用户定义的运算符只是作为普通函数构建的（因为“原型”归结为具有正确名称的函数）所有内容都已落实到位。

> 我们遗漏的最后一段代码是一些顶级魔法：
```cpp
Function *FunctionAST::codegen() {
  // Transfer ownership of the prototype to the FunctionProtos map, but keep a
  // reference to it for use below.
  auto &P = *Proto;
  FunctionProtos[Proto->getName()] = std::move(Proto);
  Function *TheFunction = getFunction(P.getName());
  if (!TheFunction)
    return nullptr;

  // If this is an operator, install it.
  if (P.isBinaryOp())
    BinopPrecedence[P.getOperatorName()] = P.getBinaryPrecedence();

  // Create a new basic block to start insertion into.
  BasicBlock *BB = BasicBlock::Create(TheContext, "entry", TheFunction);
  ...
```
> 基本上，在对代码进行代码化之前，如果它是用户定义的运算符，我们会在优先级表中注册它。这允许二元运算符解析我们已有的逻辑来处理它。由于我们正在研究一个完全通用的运算符优先级解析器，所以我们需要做的就是“扩展语法”。

> 现在我们有了有用的用户定义二元运算符。这构建了我们为其他运营商构建的先前框架。添加一元运算符更具挑战性，因为我们还没有任何框架 - 让我们看看它需要什么。

## 6.4 用户定义的一元运算符
> 由于我们目前不支持Kaleidoscope语言中的一元运算符，因此我们需要添加所有内容以支持它们。上面，我们为词法分析器添加了对“一元”关键字的简单支持。除此之外，我们需要一个AST节点：

```cpp
/// UnaryExprAST - Expression class for a unary operator.
class UnaryExprAST : public ExprAST {
  char Opcode;
  std::unique_ptr<ExprAST> Operand;

public:
  UnaryExprAST(char Opcode, std::unique_ptr<ExprAST> Operand)
    : Opcode(Opcode), Operand(std::move(Operand)) {}

  Value *codegen() override;
};
```
> 到目前为止，这个AST节点非常简单明了。它直接镜像二元运算符AST节点，除了它只有一个子节点。有了这个，我们需要添加解析逻辑。解析一元运算符非常简单：我们将添加一个新函数来执行此操作：
```cpp
/// unary
///   ::= primary
///   ::= '!' unary
static std::unique_ptr<ExprAST> ParseUnary() {
  // If the current token is not an operator, it must be a primary expr.
  if (!isascii(CurTok) || CurTok == '(' || CurTok == ',')
    return ParsePrimary();

  // If this is a unary operator, read it.
  int Opc = CurTok;
  getNextToken();
  if (auto Operand = ParseUnary())
    return llvm::make_unique<UnaryExprAST>(Opc, std::move(Operand));
  return nullptr;
}
```
> 我们添加的语法非常简单。如果我们在解析主运算符时看到一元运算符，我们将运算符作为前缀并将剩余的块解析为另一个一元运算符。这允许我们处理多个一元运算符（例如“!! x”）。请注意，一元运算符不能像二元运算符那样具有模糊的解析，因此不需要优先级信息。

> 这个函数的问题是我们需要从某个地方调用ParseUnary。为此，我们改变ParsePrimary的先前调用者来调用ParseUnary：

```cpp
/// binoprhs
///   ::= ('+' unary)*
static std::unique_ptr<ExprAST> ParseBinOpRHS(int ExprPrec,
                                              std::unique_ptr<ExprAST> LHS) {
  ...
    // Parse the unary expression after the binary operator.
    auto RHS = ParseUnary();
    if (!RHS)
      return nullptr;
  ...
}
/// expression
///   ::= unary binoprhs
///
static std::unique_ptr<ExprAST> ParseExpression() {
  auto LHS = ParseUnary();
  if (!LHS)
    return nullptr;

  return ParseBinOpRHS(0, std::move(LHS));
}
```
> 通过这两个简单的更改，我们现在能够解析一元运算符并为它们构建AST。接下来，我们需要为原型添加解析器支持，以解析一元运算符原型。我们将上面的二元运算符代码扩展为：
```cpp
/// prototype
///   ::= id '(' id* ')'
///   ::= binary LETTER number? (id, id)
///   ::= unary LETTER (id)
static std::unique_ptr<PrototypeAST> ParsePrototype() {
  std::string FnName;

  unsigned Kind = 0;  // 0 = identifier, 1 = unary, 2 = binary.
  unsigned BinaryPrecedence = 30;

  switch (CurTok) {
  default:
    return LogErrorP("Expected function name in prototype");
  case tok_identifier:
    FnName = IdentifierStr;
    Kind = 0;
    getNextToken();
    break;
  case tok_unary:
    getNextToken();
    if (!isascii(CurTok))
      return LogErrorP("Expected unary operator");
    FnName = "unary";
    FnName += (char)CurTok;
    Kind = 1;
    getNextToken();
    break;
  case tok_binary:
    ...
```
> 与二元运算符一样，我们将一元运算符命名为包含运算符字符的名称。这有助于我们在代码生成时间。说到，我们需要添加的最后一部分是对一元运算符的codegen支持。它看起来像这样：
```cpp
Value *UnaryExprAST::codegen() {
  Value *OperandV = Operand->codegen();
  if (!OperandV)
    return nullptr;

  Function *F = getFunction(std::string("unary") + Opcode);
  if (!F)
    return LogErrorV("Unknown unary operator");

  return Builder.CreateCall(F, OperandV, "unop");
}
此代码与二进制运算符的代码类似，但更简单。它更简单，主要是因为它不需要处理任何预定义的运算符。
```
## 6.5 造轮子
> 这有点难以置信，但是我们已经在最后几章中介绍了一些简单的扩展，我们已经成长为一种真正的语言。有了这个，我们可以做很多有趣的事情，包括I / O，数学和其他一些东西。例如，我们现在可以添加一个很好的排序运算符（printd被定义为打印出指定的值和换行符）：
```sh
ready> extern printd(x);
Read extern:
declare double @printd(double)

ready> def binary : 1 (x y) 0;  # Low-precedence operator that ignores operands.
...
ready> printd(123) : printd(456) : printd(789);
123.000000
456.000000
789.000000
Evaluated to 0.000000
```
> 我们还可以定义一堆其他“原始”操作，例如：
```sh
# Logical unary not.
def unary!(v)
  if v then
    0
  else
    1;

# Unary negate.
def unary-(v)
  0-v;

# Define > with the same precedence as <.
def binary> 10 (LHS RHS)
  RHS < LHS;

# Binary logical or, which does not short circuit.
def binary| 5 (LHS RHS)
  if LHS then
    1
  else if RHS then
    1
  else
    0;

# Binary logical and, which does not short circuit.
def binary& 6 (LHS RHS)
  if !LHS then
    0
  else
    !!RHS;

# Define = with slightly lower precedence than relationals.
def binary = 9 (LHS RHS)
  !(LHS < RHS | LHS > RHS);

# Define ':' for sequencing: as a low-precedence operator that ignores operands
# and just returns the RHS.
def binary : 1 (x y) y;
```
> 鉴于之前的if / then / else支持，我们还可以为I / O定义有趣的函数。例如，下面打印出一个字符，其“密度”反映传入的值：值越低，字符越密集：
```sh
ready> extern putchard(char);
...
ready> def printdensity(d)
  if d > 8 then
    putchard(32)  # ' '
  else if d > 4 then
    putchard(46)  # '.'
  else if d > 2 then
    putchard(43)  # '+'
  else
    putchard(42); # '*'
...
ready> printdensity(1): printdensity(2): printdensity(3):
       printdensity(4): printdensity(5): printdensity(9):
       putchard(10);
**++.
Evaluated to 0.000000
```
> 基于这些简单的原始操作，我们可以开始定义更有趣的事情。例如，这是一个小函数，它确定复平面中某个函数发散所需的迭代次数：
```sh
# Determine whether the specific location diverges.
# Solve for z = z^2 + c in the complex plane.
def mandelconverger(real imag iters creal cimag)
  if iters > 255 | (real*real + imag*imag > 4) then
    iters
  else
    mandelconverger(real*real - imag*imag + creal,
                    2*real*imag + cimag,
                    iters+1, creal, cimag);

# Return the number of iterations required for the iteration to escape
def mandelconverge(real imag)
  mandelconverger(real, imag, 0, real, imag);
```
> 这个“ ”功能是一个美丽的小动物，是计算Mandelbrot Set的基础。我们的 函数返回复杂轨道逃逸所需的迭代次数，饱和到255.这本身并不是一个非常有用的函数，但是如果你在二维平面上绘制它的值，你可以看到Mandelbrot集合。鉴于我们仅限于在这里使用putchard，我们惊人的图形输出是有限的，但我们可以使用上面的密度绘图仪鞭打一些东西：z = z2 + cmandelconverge
```sh
# Compute and plot the mandelbrot set with the specified 2 dimensional range
# info.
def mandelhelp(xmin xmax xstep   ymin ymax ystep)
  for y = ymin, y < ymax, ystep in (
    (for x = xmin, x < xmax, xstep in
       printdensity(mandelconverge(x,y)))
    : putchard(10)
  )

# mandel - This is a convenient helper function for plotting the mandelbrot set
# from the specified position with the specified Magnification.
def mandel(realstart imagstart realmag imagmag)
  mandelhelp(realstart, realstart+realmag*78, realmag,
             imagstart, imagstart+imagmag*40, imagmag);
```
鉴于此，我们可以尝试绘制出mandelbrot集！让我们尝试一下：
```sh
ready> mandel(-2.3, -1.3, 0.05, 0.07);
*******************************+++++++++++*************************************
*************************+++++++++++++++++++++++*******************************
**********************+++++++++++++++++++++++++++++****************************
*******************+++++++++++++++++++++.. ...++++++++*************************
*****************++++++++++++++++++++++.... ...+++++++++***********************
***************+++++++++++++++++++++++.....   ...+++++++++*********************
**************+++++++++++++++++++++++....     ....+++++++++********************
*************++++++++++++++++++++++......      .....++++++++*******************
************+++++++++++++++++++++.......       .......+++++++******************
***********+++++++++++++++++++....                ... .+++++++*****************
**********+++++++++++++++++.......                     .+++++++****************
*********++++++++++++++...........                    ...+++++++***************
********++++++++++++............                      ...++++++++**************
********++++++++++... ..........                        .++++++++**************
*******+++++++++.....                                   .+++++++++*************
*******++++++++......                                  ..+++++++++*************
*******++++++.......                                   ..+++++++++*************
*******+++++......                                     ..+++++++++*************
*******.... ....                                      ...+++++++++*************
*******.... .                                         ...+++++++++*************
*******+++++......                                    ...+++++++++*************
*******++++++.......                                   ..+++++++++*************
*******++++++++......                                   .+++++++++*************
*******+++++++++.....                                  ..+++++++++*************
********++++++++++... ..........                        .++++++++**************
********++++++++++++............                      ...++++++++**************
*********++++++++++++++..........                     ...+++++++***************
**********++++++++++++++++........                     .+++++++****************
**********++++++++++++++++++++....                ... ..+++++++****************
***********++++++++++++++++++++++.......       .......++++++++*****************
************+++++++++++++++++++++++......      ......++++++++******************
**************+++++++++++++++++++++++....      ....++++++++********************
***************+++++++++++++++++++++++.....   ...+++++++++*********************
*****************++++++++++++++++++++++....  ...++++++++***********************
*******************+++++++++++++++++++++......++++++++*************************
*********************++++++++++++++++++++++.++++++++***************************
*************************+++++++++++++++++++++++*******************************
******************************+++++++++++++************************************
*******************************************************************************
*******************************************************************************
*******************************************************************************
Evaluated to 0.000000
ready> mandel(-2, -1, 0.02, 0.04);
**************************+++++++++++++++++++++++++++++++++++++++++++++++++++++
***********************++++++++++++++++++++++++++++++++++++++++++++++++++++++++
*********************+++++++++++++++++++++++++++++++++++++++++++++++++++++++++.
*******************+++++++++++++++++++++++++++++++++++++++++++++++++++++++++...
*****************+++++++++++++++++++++++++++++++++++++++++++++++++++++++++.....
***************++++++++++++++++++++++++++++++++++++++++++++++++++++++++........
**************++++++++++++++++++++++++++++++++++++++++++++++++++++++...........
************+++++++++++++++++++++++++++++++++++++++++++++++++++++..............
***********++++++++++++++++++++++++++++++++++++++++++++++++++........        .
**********++++++++++++++++++++++++++++++++++++++++++++++.............
********+++++++++++++++++++++++++++++++++++++++++++..................
*******+++++++++++++++++++++++++++++++++++++++.......................
******+++++++++++++++++++++++++++++++++++...........................
*****++++++++++++++++++++++++++++++++............................
*****++++++++++++++++++++++++++++...............................
****++++++++++++++++++++++++++......   .........................
***++++++++++++++++++++++++.........     ......    ...........
***++++++++++++++++++++++............
**+++++++++++++++++++++..............
**+++++++++++++++++++................
*++++++++++++++++++.................
*++++++++++++++++............ ...
*++++++++++++++..............
*+++....++++................
*..........  ...........
*
*..........  ...........
*+++....++++................
*++++++++++++++..............
*++++++++++++++++............ ...
*++++++++++++++++++.................
**+++++++++++++++++++................
**+++++++++++++++++++++..............
***++++++++++++++++++++++............
***++++++++++++++++++++++++.........     ......    ...........
****++++++++++++++++++++++++++......   .........................
*****++++++++++++++++++++++++++++...............................
*****++++++++++++++++++++++++++++++++............................
******+++++++++++++++++++++++++++++++++++...........................
*******+++++++++++++++++++++++++++++++++++++++.......................
********+++++++++++++++++++++++++++++++++++++++++++..................
Evaluated to 0.000000
ready> mandel(-0.9, -1.4, 0.02, 0.03);
*******************************************************************************
*******************************************************************************
*******************************************************************************
**********+++++++++++++++++++++************************************************
*+++++++++++++++++++++++++++++++++++++++***************************************
+++++++++++++++++++++++++++++++++++++++++++++**********************************
++++++++++++++++++++++++++++++++++++++++++++++++++*****************************
++++++++++++++++++++++++++++++++++++++++++++++++++++++*************************
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++**********************
+++++++++++++++++++++++++++++++++.........++++++++++++++++++*******************
+++++++++++++++++++++++++++++++....   ......+++++++++++++++++++****************
+++++++++++++++++++++++++++++.......  ........+++++++++++++++++++**************
++++++++++++++++++++++++++++........   ........++++++++++++++++++++************
+++++++++++++++++++++++++++.........     ..  ...+++++++++++++++++++++**********
++++++++++++++++++++++++++...........        ....++++++++++++++++++++++********
++++++++++++++++++++++++.............       .......++++++++++++++++++++++******
+++++++++++++++++++++++.............        ........+++++++++++++++++++++++****
++++++++++++++++++++++...........           ..........++++++++++++++++++++++***
++++++++++++++++++++...........                .........++++++++++++++++++++++*
++++++++++++++++++............                  ...........++++++++++++++++++++
++++++++++++++++...............                 .............++++++++++++++++++
++++++++++++++.................                 ...............++++++++++++++++
++++++++++++..................                  .................++++++++++++++
+++++++++..................                      .................+++++++++++++
++++++........        .                               .........  ..++++++++++++
++............                                         ......    ....++++++++++
..............                                                    ...++++++++++
..............                                                    ....+++++++++
..............                                                    .....++++++++
.............                                                    ......++++++++
...........                                                     .......++++++++
.........                                                       ........+++++++
.........                                                       ........+++++++
.........                                                           ....+++++++
........                                                             ...+++++++
.......                                                              ...+++++++
                                                                    ....+++++++
                                                                   .....+++++++
                                                                    ....+++++++
                                                                    ....+++++++
                                                                    ....+++++++
Evaluated to 0.000000
ready> ^D
```
> 此时，您可能已经开始意识到Kaleidoscope是一种真实而强大的语言。它可能不是自相似的:)，但它可以用于绘制事物！

> 有了这个，我们总结了本教程的“添加用户定义的运算符”一章。我们已经成功地扩充了我们的语言，添加了扩展库中语言的能力，并且我们已经展示了如何在Kaleidoscope中构建一个简单但有趣的最终用户应用程序。此时，Kaleidoscope可以构建各种功能性的应用程序，并且可以调用具有副作用的函数，但它实际上无法定义和变异变量本身。

> 引人注目的是，变量变异是某些语言的一个重要特征，如何在不必向前端添加“SSA构造”阶段的情况下添加对可变变量的支持并不是很明显。在下一章中，我们将介绍如何在不在前端构建SSA的情况下添加变量变异。

## 6.6 完整的代码清单
以下是我们运行示例的完整代码清单，增强了对用户定义运算符的支持。要构建此示例，请使用：
```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core mcjit native` -O3 -o toy
# Run
./toy
```
> 在某些平台上，您需要在链接时指定-rdynamic或-Wl，-export-dynamic。这可确保主可执行文件中定义的符号导出到动态链接器，因此可在运行时进行符号解析。如果将支持代码编译到共享库中，则不需要这样做，但这样做会导致Windows出现问题。

> 这是代码：[toy.cpp](./toy.cpp)


[下一页：扩展语言：可变变量/ SSA构造](../Chapter07/README.md)