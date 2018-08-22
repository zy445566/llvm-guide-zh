# 3.万花筒：代码生成LLVM IR 
* 第3章简介
* 代码生成设置
* 表达代码生成
* 功能代码生成
* 驱动变动与结束思考
* 完整的代码清单
## 3.1 第3章介绍
> 欢迎阅读“ 使用LLVM实现语言 ”教程的第3章。本章介绍如何将第2章中构建的抽象语法树转换为LLVM IR。这将教你一些关于LLVM如何做事的内容，以及演示它的易用性。构建词法分析器和解析器要比生成LLVM IR代码要多得多。:)

> 请注意：本章和后面的代码需要LLVM 3.7或更高版本。LLVM 3.6及之前的版本不适用。另请注意，您需要使用与LLVM版本匹配的本教程版本：如果您使用的是正式的LLVM版本，请使用您的版本附带的文档版本或llvm.org版本页面。

##3.2 代码生成设置
> 为了生成LLVM IR，我们需要一些简单的设置才能开始。首先，我们在每个AST类中定义虚拟代码生成（codegen）方法：
```cpp
/// ExprAST - Base class for all expression nodes.
class ExprAST {
public:
  virtual ~ExprAST() {}
  virtual Value *codegen() = 0;
};

/// NumberExprAST - Expression class for numeric literals like "1.0".
class NumberExprAST : public ExprAST {
  double Val;

public:
  NumberExprAST(double Val) : Val(Val) {}
  virtual Value *codegen();
};
...
```
> codegen（）方法表示为该AST节点发出IR及其依赖的所有内容，并且它们都返回一个LLVM Value对象。“Value”是用于表示 LLVM中的“ 静态单一分配（SSA）寄存器”或“SSA值”的类。SSA值的最独特之处在于它们的值是在相关指令执行时计算的，并且在指令重新执行之前（以及如果）它不会获得新值。换句话说，没有办法“改变”SSA值。欲了解更多信息，请阅读静态单一作业 - 一旦你理解它们，这些概念就非常自然了。

> 请注意，不是将虚方法添加到ExprAST类层次结构中，而是使用访问者模式或其他方式对此进行建模也是有意义的。同样，本教程将不再讨论良好的软件工程实践：出于我们的目的，添加虚拟方法是最简单的。

> 我们想要的第二件事是像我们用于解析器的“LogError”方法，它将用于报告在代码生成期间发现的错误（例如，使用未声明的参数）：
```cpp
static LLVMContext TheContext;
static IRBuilder<> Builder(TheContext);
static std::unique_ptr<Module> TheModule;
static std::map<std::string, Value *> NamedValues;

Value *LogErrorV(const char *Str) {
  LogError(Str);
  return nullptr;
}
```
> 静态变量将在代码生成期间使用。TheContext 是一个不透明的对象，拥有许多核心LLVM数据结构，例如类型和常量值表。我们不需要详细了解它，我们只需要一个实例就可以传递到需要它的API中。

> 该Builder对象是一个辅助对象，可以轻松生成LLVM指令。IRBuilder 类模板的实例 跟踪插入指令的当前位置，并具有创建新指令的方法。

> TheModule是一个包含函数和全局变量的LLVM构造。在许多方面，它是LLVM IR用于包含代码的顶级结构。它将拥有我们生成的所有IR的内存，这就是codegen（）方法返回原始值*而不是unique_ptr <Value>的原因。

> 该NamedValues地图跟踪哪些值在当前范围以及他们LLVM表示被定义。（换句话说，它是代码的符号表）。在这种形式的万花筒中，唯一可以引用的是功能参数。因此，在为其函数体生成代码时，函数参数将在此映射中。

> 有了这些基础知识，我们就可以开始讨论如何为每个表达式生成代码。请注意，这个假设Builder 已经设置来生成代码到一些东西。现在，我们假设已经完成了，我们只是用它来发出代码。

## 3.3 表达式代码生成
> 为表达式节点生成LLVM代码非常简单：所有四个表达式节点的注释代码少于45行。首先，我们将做数字文字：
```cpp
Value *NumberExprAST::codegen() {
  return ConstantFP::get(TheContext, APFloat(Val));
}
```
> 在LLVM IR中，数字常量用ConstantFP类表示 ，它在APFloat 内部保存数值（APFloat能够保持任意精度的浮点常量）。这段代码基本上只是创建并返回一个ConstantFP。请注意，在LLVM IR中，常量都是唯一的并且共享。出于这个原因，API使用“foo :: get（...）”成语而不是“new foo（..）”或“foo :: Create（..）”。
```cpp
Value *VariableExprAST::codegen() {
  // Look this variable up in the function.
  Value *V = NamedValues[Name];
  if (!V)
    LogErrorV("Unknown variable name");
  return V;
}
```
> 使用LLVM对变量的引用也非常简单。在简单版的Kaleidoscope中，我们假设变量已经在某处发出并且其值可用。实际上，NamedValues映射中唯一的值是函数参数。此代码只是检查指定的名称是否在映射中（如果没有，则引用未知变量）并返回其值。在以后的章节中，我们将在符号表和局部变量中添加对循环归纳变量的支持。
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
    return LogErrorV("invalid binary operator");
  }
}
```
> 二元运算符开始变得更有趣。这里的基本思想是我们递归地为表达式的左侧发出代码，然后是右侧，然后我们计算二进制表达式的结果。在此代码中，我们对操作码进行了简单的切换，以创建正确的LLVM指令。

> 在上面的示例中，LLVM构建器类开始显示其值。IRBuilder知道在哪里插入新创建的指令，您所要做的就是指定要创建的指令（例如，使用 CreateFAdd），使用哪些操作数（L以及R此处），并可选择为生成的指令提供名称。

> LLVM的一个好处是名称只是一个提示。例如，如果上面的代码发出多个“addtmp”变量，LLVM将自动为每个变量提供一个增加的唯一数字后缀。指令的本地值名称纯粹是可选的，但它使读取IR转储更加容易。

> LLVM指令受严格规则的约束：例如，add指令的Left和Right运算符必须具有相同的类型，add的结果类型必须与操作数类型匹配。因为Kaleidoscope中的所有值都是双倍的，所以这为add，sub和mul提供了非常简单的代码。

> 另一方面，LLVM指定fcmp指令始终返回'i1'值（一位整数）。这个问题是Kaleidoscope希望值为0.0或1.0。为了获得这些语义，我们将fcmp指令与uitofp指令结合起来。该指令通过将输入视为无符号值将其输入整数转换为浮点值。相反，如果我们使用sitofp指令，Kaleidoscope'<'运算符将返回0.0和-1.0，具体取决于输入值。
```cpp
Value *CallExprAST::codegen() {
  // Look up the name in the global module table.
  Function *CalleeF = TheModule->getFunction(Callee);
  if (!CalleeF)
    return LogErrorV("Unknown function referenced");

  // If argument mismatch error.
  if (CalleeF->arg_size() != Args.size())
    return LogErrorV("Incorrect # arguments passed");

  std::vector<Value *> ArgsV;
  for (unsigned i = 0, e = Args.size(); i != e; ++i) {
    ArgsV.push_back(Args[i]->codegen());
    if (!ArgsV.back())
      return nullptr;
  }

  return Builder.CreateCall(CalleeF, ArgsV, "calltmp");
}
```
> 使用LLVM时，函数调用的代码生成非常简单。上面的代码最初在LLVM模块的符号表中执行函数名称查找。回想一下，LLVM模块是容纳我们JIT的功能的容器。通过为每个函数指定与用户指定的名称相同的名称，我们可以使用LLVM符号表来解析我们的函数名称。

> 一旦我们有了要调用的函数，我们递归地对每个要传入的参数进行编码，并创建一个LLVM 调用指令。请注意，LLVM默认使用本机C调用约定，允许这些调用也调用标准库函数，如“sin”和“cos”，无需额外的工作。

> 这包含了我们对Kaleidoscope中迄今为止的四个基本表达式的处理。随意进入并添加更多。例如，通过浏览LLVM语言参考，您将找到其他一些非常容易插入基本框架的有趣指令。

## 3.4 功能代码生成
> 原型和函数的代码生成必须处理许多细节，这使得它们的代码不如表达式代码生成美观，但允许我们说明一些重要的观点。首先，我们来谈谈原型的代码生成：它们既用于函数体，也用于外部函数声明。代码以：
```cpp
Function *PrototypeAST::codegen() {
  // Make the function type:  double(double,double) etc.
  std::vector<Type*> Doubles(Args.size(),
                             Type::getDoubleTy(TheContext));
  FunctionType *FT =
    FunctionType::get(Type::getDoubleTy(TheContext), Doubles, false);

  Function *F =
    Function::Create(FT, Function::ExternalLinkage, Name, TheModule);
```
> 这段代码将很多功能集成到几行中。首先请注意，此函数返回“Function *”而不是“Value *”。因为“原型”真的是在谈论函数的外部接口（而不是由表达式计算的值），所以它返回与codegen时相对应的LLVM函数是有意义的。

> FunctionType::get创建FunctionType它的调用应该用于给定的Prototype。由于Kaleidoscope中的所有函数参数都是double类型，因此第一行创建了一个“N”LLVM double类型的向量。然后它使用该Functiontype::get方法创建一个函数类型，该函数类型将“N”双精度作为参数，结果返回一个double，而不是vararg（false参数表示这一点）。请注意，LLVM中的类型与常量一样是唯一的，所以你不要“新”一个类型，你“得到”它。

> 上面的最后一行实际上创建了与Prototype相对应的IR功能。这表示要使用的类型，链接和名称，以及要插入的模块。“ 外部链接 ”意味着该功能可以在当前模块外部定义和/或可以由模块外部的功能调用。传入的名称是用户指定的名称：由于指定了“ TheModule”，因此该名称在“ TheModule”符号表中注册。
```cpp
// Set names for all arguments.
unsigned Idx = 0;
for (auto &Arg : F->args())
  Arg.setName(Args[Idx++]);

return F;
```
> 最后，我们根据Prototype中给出的名称设置每个函数参数的名称。此步骤并非严格必要，但保持名称一致会使IR更具可读性，并允许后续代码直接引用其名称的参数，而不必在Prototype AST中查找它们。

> 在这一点上，我们有一个没有身体的功能原型。这是LLVM IR表示函数声明的方式。对于Kaleidoscope中的外部声明，这是我们需要的。但是对于函数定义，我们需要codegen并附加一个函数体。
```cpp
Function *FunctionAST::codegen() {
    // First, check for an existing function from a previous 'extern' declaration.
  Function *TheFunction = TheModule->getFunction(Proto->getName());

  if (!TheFunction)
    TheFunction = Proto->codegen();

  if (!TheFunction)
    return nullptr;

  if (!TheFunction->empty())
    return (Function*)LogErrorV("Function cannot be redefined.");
```
> 对于函数定义，我们首先在TheModule的符号表中搜索此函数的现有版本，以防已经使用'extern'语句创建了一个。如果Module :: getFunction返回null，则不存在先前版本，因此我们将从Prototype中编译一个。在任何一种情况下，我们都希望在开始之前断言函数是空的（即没有正文）。
```cpp
// Create a new basic block to start insertion into.
BasicBlock *BB = BasicBlock::Create(TheContext, "entry", TheFunction);
Builder.SetInsertPoint(BB);

// Record the function arguments in the NamedValues map.
NamedValues.clear();
for (auto &Arg : TheFunction->args())
  NamedValues[Arg.getName()] = &Arg;
```
> 现在我们到达Builder设置点。第一行创建一个新的基本块 （名为“entry”），插入其中TheFunction。然后第二行告诉构建器应该将新指令插入到新基本块的末尾。LLVM中的基本块是定义控制流图的函数的重要部分。由于我们没有任何控制流，因此我们的函数此时只包含一个块。我们将在第5章修复此问题:)。

> 接下来，我们将函数参数添加到NamedValues映射（首先清除它之后），以便VariableExprAST节点可以访问它们。
```cpp
if (Value *RetVal = Body->codegen()) {
  // Finish off the function.
  Builder.CreateRet(RetVal);

  // Validate the generated code, checking for consistency.
  verifyFunction(*TheFunction);

  return TheFunction;
}
```
> 一旦设置了插入点并填充了NamedValues映射，我们就会调用codegen()该函数的根表达式的方法。如果没有发生错误，则会发出代码以将表达式计算到条目块中并返回计算的值。假设没有错误，我们然后创建一个LLVM ret指令，完成该功能。构建函数后，我们调用verifyFunction，由LLVM提供。此函数对生成的代码执行各种一致性检查，以确定我们的编译器是否正在执行所有操作。使用它很重要：它可以捕获很多错误。功能完成并验证后，我们将其返回。
```cpp
  // Error reading body, remove function.
  TheFunction->eraseFromParent();
  return nullptr;
}
```
> 这里留下的唯一一件事是处理错误案例。为简单起见，我们仅通过删除使用该eraseFromParent方法生成的函数来处理此问题 。这允许用户重新定义之前错误输入的函数：如果我们没有删除它，它将存在于带有正文的符号表中，从而阻止将来重新定义。

> 但是，此代码确实存在错误：如果FunctionAST::codegen()方法找到现有的IR函数，则它不会根据定义自己的原型验证其签名。这意味着较早的'extern'声明将优先于函数定义的签名，这可能导致codegen失败，例如，如果函数参数的名称不同。有很多方法可以解决这个问题，看看你能想出什么！这是一个测试用例：
```sh
extern foo(a);     # ok, defines foo.
def foo(b) b;      # Error: Unknown variable name. (decl using 'a' takes precedence).
```
## 3.5 驱动变更和结束思路
目前，除了我们可以查看漂亮的IR调用之外，LLVM的代码生成并没有给我们带来太多帮助。示例代码将对codegen的调用插入到“ HandleDefinition”，“ HandleExtern”等函数中，然后转储出LLVM IR。这为查看简单函数的LLVM IR提供了一种很好的方法。例如：
```sh
ready> 4+5;
Read top-level expression:
define double @0() {
entry:
  ret double 9.000000e+00
}
```
> 请注意解析器如何将顶级表达式转换为我们的匿名函数。当我们在下一章中添加JIT支持时，这将非常方便。另请注意，代码非常精确地转录，除了IRBuilder完成的简单常量折叠之外，不会执行任何优化。我们将在下一章中明确添加优化。
```sh 
ready> def foo(a b) a*a + 2*a*b + b*b;
Read function definition:
define double @foo(double %a, double %b) {
entry:
  %multmp = fmul double %a, %a
  %multmp1 = fmul double 2.000000e+00, %a
  %multmp2 = fmul double %multmp1, %b
  %addtmp = fadd double %multmp, %multmp2
  %multmp3 = fmul double %b, %b
  %addtmp4 = fadd double %addtmp, %multmp3
  ret double %addtmp4
}
```
> 这显示了一些简单的算术。请注意与我们用于创建指令的LLVM构建器调用具有惊人的相似性。
```sh
ready> def bar(a) foo(a, 4.0) + bar(31337);
Read function definition:
define double @bar(double %a) {
entry:
  %calltmp = call double @foo(double %a, double 4.000000e+00)
  %calltmp1 = call double @bar(double 3.133700e+04)
  %addtmp = fadd double %calltmp, %calltmp1
  ret double %addtmp
}
```
> 这显示了一些函数调用。请注意，如果您调用此函数将需要很长时间才能执行。在未来我们将添加条件控制流以实际使递归有用:)。
```cpp
ready> extern cos(x);
Read extern:
declare double @cos(double)

ready> cos(1.234);
Read top-level expression:
define double @1() {
entry:
  %calltmp = call double @cos(double 1.234000e+00)
  ret double %calltmp
}
```
> 这显示了libm“cos”函数的extern，以及对它的调用。
```sh
ready> ^D
; ModuleID = 'my cool jit'

define double @0() {
entry:
  %addtmp = fadd double 4.000000e+00, 5.000000e+00
  ret double %addtmp
}

define double @foo(double %a, double %b) {
entry:
  %multmp = fmul double %a, %a
  %multmp1 = fmul double 2.000000e+00, %a
  %multmp2 = fmul double %multmp1, %b
  %addtmp = fadd double %multmp, %multmp2
  %multmp3 = fmul double %b, %b
  %addtmp4 = fadd double %addtmp, %multmp3
  ret double %addtmp4
}

define double @bar(double %a) {
entry:
  %calltmp = call double @foo(double %a, double 4.000000e+00)
  %calltmp1 = call double @bar(double 3.133700e+04)
  %addtmp = fadd double %calltmp, %calltmp1
  ret double %addtmp
}

declare double @cos(double)

define double @1() {
entry:
  %calltmp = call double @cos(double 1.234000e+00)
  ret double %calltmp
}
```
> 当您退出当前演示时（通过在Linux上通过CTRL + D发送EOF或在Windows上按CTRL + Z和ENTER发送EOF），它会为生成的整个模块转储IR。在这里，您可以看到所有功能相互引用的大图。

> 这包含了Kaleidoscope教程的第三章。接下来，我们将描述如何为此添加JIT codegen和优化器支持，以便我们实际上可以开始运行代码！

## 3.6 完整的代码清单
> 以下是我们运行示例的完整代码清单，使用LLVM代码生成器进行了增强。因为它使用LLVM库，我们需要将它们链接起来。为此，我们使用 llvm-config工具通知makefile /命令行有关使用哪些选项：
```sh
# Compile
clang++ -g -O3 toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core` -o toy
# Run
./toy
```
> 这是代码：[toy.cpp](./toy.cpp)

[下一页：添加JIT和优化程序支持](../Chapter4/README.md)