# 7.万花筒：扩展语言：可变变量
* 第7章简介
* 为什么这是一个难题？
* LLVM中的内存
* 万花筒中的可变变量
* 调整现有变量以进行变异
* 新任务运营商
* 用户定义的局部变量
* 完整的代码清单
## 7.1 第7章介绍
> 欢迎阅读“ 使用LLVM实现语言 ”教程的第7章。在第1章到第6章中，我们构建了一种非常受欢迎的，虽然简单，功能强大的编程语言。在我们的旅程中，我们学习了一些解析技术，如何构建和表示AST，如何构建LLVM IR，以及如何优化结果代码以及JIT编译它。

> 虽然Kaleidoscope作为一种功能性语言很有意思，但它的功能性使得为它生成LLVM IR“太容易”了。特别是，函数式语言使得直接以SSA形式构建LLVM IR变得非常容易。由于LLVM要求输入代码采用SSA格式，因此这是一个非常好的属性，新手通常不清楚如何使用可变变量为命令式语言生成代码。

> 本章的简短（和快乐）摘要是，您的前端无需构建SSA表单：LLVM为此提供了高度优化且经过良好测试的支持，尽管它的工作方式对于某些人来说有点意外。

## 7.2 为什么这是一个难题？
> 要理解为什么可变变量会导致SSA构造的复杂性，请考虑这个非常简单的C示例：
```sh
int G, H;
int test(_Bool Condition) {
  int X;
  if (Condition)
    X = G;
  else
    X = H;
  return X;
}
```
> 在这种情况下，我们有变量“X”，其值取决于程序中执行的路径。因为在返回指令之前X有两个不同的可能值，所以插入PHI节点以合并这两个值。我们在这个例子中想要的LLVM IR如下所示：
```sh
@G = weak global i32 0   ; type of @G is i32*
@H = weak global i32 0   ; type of @H is i32*

define i32 @test(i1 %Condition) {
entry:
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32* @G
  br label %cond_next

cond_false:
  %X.1 = load i32* @H
  br label %cond_next

cond_next:
  %X.2 = phi i32 [ %X.1, %cond_false ], [ %X.0, %cond_true ]
  ret i32 %X.2
}
```
> 在此示例中，来自G和H全局变量的加载在LLVM IR中是显式的，并且它们位于if语句的then / else分支中（cond_true / cond_false）。为了合并传入的值，cond_next块中的X.2 phi节点根据控制流来自何处来选择要使用的正确值：如果控制流来自cond_false块，则X.2获取X的值0.1。或者，如果控制流来自cond_true，则它获得X.0的值。本章的目的不是解释SSA表格的细节。有关更多信息，请参阅众多在线参考资料之一。

> 本文的问题是“在降低对可变变量的赋值时，谁将phi节点放置？”。这里的问题是LLVM 要求其IR处于SSA形式：它没有“非ssa”模式。然而，SSA构造需要非平凡的算法和数据结构，因此每个前端必须重现这种逻辑是不方便和浪费的。

## 7.3 LLVM中的内存
> 这里的“技巧”是，虽然LLVM确实要求所有寄存器值都是SSA形式，但它不要求（或允许）存储器对象采用SSA形式。在上面的示例中，请注意G和H的负载是对G和H的直接访问：它们不会重命名或版本化。这与其他一些尝试版本内存对象的编译器系统不同。在LLVM中，不是将内存的数据流分析编码到LLVM IR中，而是使用按需计算的Analysis Passes进行处理。

> 考虑到这一点，高级想法是我们想要为函数中的每个可变对象创建一个堆栈变量（它存在于内存中，因为它在堆栈中）。为了利用这个技巧，我们需要讨论LLVM如何表示堆栈变量。

> 在LLVM中，所有内存访问都是使用加载/存储指令显式的，并且经过精心设计，不具备（或需要）“address-of”运算符。注意@ G / @ H全局变量的类型实际上是“i32 *”，即使变量定义为“i32”。这意味着@G 在全局数据区域中为i32 定义了空间，但其 名称实际上是指该空间的地址。堆栈变量以相同的方式工作，除了使用LLVM alloca指令声明它们而不是使用全局变量定义声明它们之外：
```sh
define i32 @example() {
entry:
  %X = alloca i32           ; type of %X is i32*.
  ...
  %tmp = load i32* %X       ; load the stack value %X from the stack.
  %tmp2 = add i32 %tmp, 1   ; increment it
  store i32 %tmp2, i32* %X  ; store it back
  ...
```
> 此代码显示了如何在LLVM IR中声明和操作堆栈变量的示例。使用alloca指令分配的堆栈内存是完全通用的：您可以将堆栈槽的地址传递给函数，您可以将其存储在其他变量中等。在上面的示例中，我们可以重写示例以使用alloca技术来避免使用PHI节点：
```sh
@G = weak global i32 0   ; type of @G is i32*
@H = weak global i32 0   ; type of @H is i32*

define i32 @test(i1 %Condition) {
entry:
  %X = alloca i32           ; type of %X is i32*.
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32* @G
  store i32 %X.0, i32* %X   ; Update X
  br label %cond_next

cond_false:
  %X.1 = load i32* @H
  store i32 %X.1, i32* %X   ; Update X
  br label %cond_next

cond_next:
  %X.2 = load i32* %X       ; Read X
  ret i32 %X.2
}
```
> 有了这个，我们发现了一种处理任意可变变量的方法，而不需要创建Phi节点：

> 每个可变变量都成为堆栈分配。
> 每次读取变量都会成为堆栈的负载。
> 变量的每次更新都成为堆栈的存储。
> 获取变量的地址只是直接使用堆栈地址。
> 虽然这个解决方案解决了我们当前的问题，但它引入了另一个问题：我们现在显然已经为非常简单和常见的操作引入了大量的堆栈流量，这是一个主要的性能问题。对我们来说幸运的是，LLVM优化器有一个名为“mem2reg”的高度优化的优化过程，可以处理这种情况，将这样的分配提升到SSA寄存器中，并根据需要插入Phi节点。例如，如果您通过传递运行此示例，您将获得：
```sh
$ llvm-as < example.ll | opt -mem2reg | llvm-dis
@G = weak global i32 0
@H = weak global i32 0

define i32 @test(i1 %Condition) {
entry:
  br i1 %Condition, label %cond_true, label %cond_false

cond_true:
  %X.0 = load i32* @G
  br label %cond_next

cond_false:
  %X.1 = load i32* @H
  br label %cond_next

cond_next:
  %X.01 = phi i32 [ %X.1, %cond_false ], [ %X.0, %cond_true ]
  ret i32 %X.01
}
```
> mem2reg传递实现了用于构造SSA形式的标准“迭代优势边界”算法，并且具有许多加速（非常常见）简并情况的优化。mem2reg优化传递是处理可变变量的答案，我们强烈建议您依赖它。请注意，mem2reg仅适用于某些情况下的变量：

> mem2reg是alloca驱动的：它查找allocas，如果它可以处理它们，它会提升它们。它不适用于全局变量或堆分配。
> mem2reg只在函数的入口块中查找alloca指令。在入口块中保证alloca只执行一次，这使得分析更简单。
> mem2reg只提升其使用是直接加载和存储的allocas。如果将堆栈对象的地址传递给函数，或者涉及任何有趣的指针算法，则不会提升alloca。
> mem2reg仅适用于第一类值的分配（例如指针，标量和向量），并且仅当分配的数组大小为1（或.ll文件中缺失）时才有效。mem2reg无法将结构或数组提升为寄存器。请注意，“sroa”传递更强大，并且在许多情况下可以提升结构，“联合”和数组。
> 所有这些属性都很容易满足大多数命令式语言，我们将在下面用Kaleidoscope进行说明。你可能会问的最后一个问题是：我应该为我的前端烦恼吗？如果我直接进行SSA构建，避免使用mem2reg优化传递会不会更好？简而言之，我们强烈建议您使用此技术来构建SSA表单，除非有非常好的理由不这样做。使用这种技术是：

> 经验证且经过充分测试：clang将此技术用于本地可变变量。因此，LLVM最常见的客户端使用它来处理大量变量。您可以确保快速找到错误并尽早修复。
> 速度极快：mem2reg有许多特殊情况，可以在常见情况下快速完成。例如，它具有仅用于单个块的变量的快速路径，仅具有一个分配点的变量，避免插入不需要的phi节点的良好启发式等。
> 调试信息生成需要：LLVM中的调试信息依赖于公开变量的地址，以便可以将调试信息附加到它。这种技术与这种调试信息风格非常吻合。
> 如果不出意外，这样可以更轻松地启动和运行前端，并且实现起来非常简单。让我们现在用可变变量扩展Kaleidoscope！

## 7.4 万花筒中的可变变量
> 现在我们知道了我们想要解决的问题，让我们看看在我们的小万花筒语言的背景下它的样子。我们将添加两个功能：

> 使用'='运算符变换变量的能力。
> 定义新变量的能力。
> 虽然第一项实际上是关于这一点的，但我们只有传入参数和归纳变量的变量，并且重新定义那些只是到目前为止:)。此外，无论您是否要改变它们，定义新变量的能力都是有用的。这是一个激励性的例子，展示了我们如何使用这些：
```sh
# Define ':' for sequencing: as a low-precedence operator that ignores operands
# and just returns the RHS.
def binary : 1 (x y) y;

# Recursive fib, we could do this before.
def fib(x)
  if (x < 3) then
    1
  else
    fib(x-1)+fib(x-2);

# Iterative fib.
def fibi(x)
  var a = 1, b = 1, c in
  (for i = 3, i < x in
     c = a + b :
     a = b :
     b = c) :
  b;

# Call it.
fibi(10);
```
> 为了改变变量，我们必须改变现有的变量来使用“alloca技巧”。一旦我们有了这个，我们将添加我们的新运算符，然后扩展Kaleidoscope以支持新的变量定义。

## 7.5 调整现有变量以进行变异
> Kaleidoscope中的符号表由代码生成时由NamedValues“地图”管理。此映射当前跟踪LLVM“Value *”，其中包含指定变量的double值。为了支持变异，我们需要稍微改变它，以便 NamedValues保存变量的内存位置。请注意，此更改是重构：它更改了代码的结构，但不（通过自身）更改编译器的行为。所有这些更改都在Kaleidoscope代码生成器中隔离。

> 在Kaleidoscope的开发中，它只支持两个变量：函数的传入参数和'for'循环的归纳变量。为了> 保持一致性，除了其他用户定义的变量外，我们还允许对这些变量进行变异。这意味着这些都需要内存位置。

> 要开始我们的Kaleidoscope转换，我们将更改NamedValues映射，使其映射到AllocaInst *而不是Value *。一旦我们这样做，C ++编译器将告诉我们需要更新的代码部分：
```cpp
static std::map<std::string, AllocaInst*> NamedValues;
此外，由于我们需要创建这些allocas，我​​们将使用一个辅助函数来确保在函数的入口块中创建allocas：

/// CreateEntryBlockAlloca - Create an alloca instruction in the entry block of
/// the function.  This is used for mutable variables etc.
static AllocaInst *CreateEntryBlockAlloca(Function *TheFunction,
                                          const std::string &VarName) {
  IRBuilder<> TmpB(&TheFunction->getEntryBlock(),
                 TheFunction->getEntryBlock().begin());
  return TmpB.CreateAlloca(Type::getDoubleTy(TheContext), 0,
                           VarName.c_str());
}
```
> 这个看起来很滑稽的代码创建了一个IRBuilder对象，它指向入口块的第一条指令（.begin（））。然后它创建一个具有预期名称的alloca并返回它。因为Kaleidoscope中的所有值都是双精度数，所以无需传入要使用的类型。

> 有了这个，我们想要做的第一个功能改变属于变量引用。在我们的新方案中，变量存在于堆栈中，因此生成对它们的引用的代码实际上需要从堆栈槽生成负载：
```cpp
Value *VariableExprAST::codegen() {
  // Look this variable up in the function.
  Value *V = NamedValues[Name];
  if (!V)
    return LogErrorV("Unknown variable name");

  // Load the value.
  return Builder.CreateLoad(V, Name.c_str());
}
```
> 如您所见，这非常简单。现在我们需要更新定义变量的内容以设置alloca。我们将从ForExprAST::codegen()（请参阅完整代码列表中的完整代码）开始：
```cpp
Function *TheFunction = Builder.GetInsertBlock()->getParent();

// Create an alloca for the variable in the entry block.
AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);

// Emit the start code first, without 'variable' in scope.
Value *StartVal = Start->codegen();
if (!StartVal)
  return nullptr;

// Store the value into the alloca.
Builder.CreateStore(StartVal, Alloca);
...

// Compute the end condition.
Value *EndCond = End->codegen();
if (!EndCond)
  return nullptr;

// Reload, increment, and restore the alloca.  This handles the case where
// the body of the loop mutates the variable.
Value *CurVar = Builder.CreateLoad(Alloca);
Value *NextVar = Builder.CreateFAdd(CurVar, StepVal, "nextvar");
Builder.CreateStore(NextVar, Alloca);
...
```
> 在我们允许可变变量之前，此代码实际上与代码相同。最大的区别是我们不再需要构建一个PHI节点，我们使用load / store来根据需要访问变量。

> 为了支持可变参数变量，我们还需要为它们进行分配。这个代码也非常简单：
```cpp
Function *FunctionAST::codegen() {
  ...
  Builder.SetInsertPoint(BB);

  // Record the function arguments in the NamedValues map.
  NamedValues.clear();
  for (auto &Arg : TheFunction->args()) {
    // Create an alloca for this variable.
    AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, Arg.getName());

    // Store the initial value into the alloca.
    Builder.CreateStore(&Arg, Alloca);

    // Add arguments to variable symbol table.
    NamedValues[Arg.getName()] = Alloca;
  }

  if (Value *RetVal = Body->codegen()) {
    ...
```
> 对于每个参数，我们创建一个alloca，将输入值存储到alloca中，并将alloca注册为参数的内存位置。FunctionAST::codegen() 在设置函数的入口块之后立即调用此方法。

> 最后遗漏的部分是添加mem2reg传递，这使我们能够再次获得良好的codegen：
```cpp
// Promote allocas to registers.
TheFPM->add(createPromoteMemoryToRegisterPass());
// Do simple "peephole" optimizations and bit-twiddling optzns.
TheFPM->add(createInstructionCombiningPass());
// Reassociate expressions.
TheFPM->add(createReassociatePass());
...
```
> 有趣的是看看mem2reg优化运行之前和之后的代码是什么样的。例如，这是我们的递归fib函数的before / after代码。在优化之前：
```sh
define double @fib(double %x) {
entry:
  %x1 = alloca double
  store double %x, double* %x1
  %x2 = load double, double* %x1
  %cmptmp = fcmp ult double %x2, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:       ; preds = %entry
  br label %ifcont

else:       ; preds = %entry
  %x3 = load double, double* %x1
  %subtmp = fsub double %x3, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %x4 = load double, double* %x1
  %subtmp5 = fsub double %x4, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ 1.000000e+00, %then ], [ %addtmp, %else ]
  ret double %iftmp
}
```
> 这里只有一个变量（x，输入参数），但你仍然可以看到我们正在使用的极其简单的代码生成策略。在输入块中，创建alloca，并将初始输入值存储到其中。每个对变量的引用都会从堆栈重新加载。另请注意，我们没有修改if / then / else表达式，因此它仍然插入了一个PHI节点。虽然我们可以为它创建一个alloca，但实际上更容易为它创建一个PHI节点，所以我们仍然只是制作PHI。

> 以下是mem2reg传递运行后的代码：
```sh
define double @fib(double %x) {
entry:
  %cmptmp = fcmp ult double %x, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:
  br label %ifcont

else:
  %subtmp = fsub double %x, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %subtmp5 = fsub double %x, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ 1.000000e+00, %then ], [ %addtmp, %else ]
  ret double %iftmp
}
```
> 这是mem2reg的一个简单案例，因为没有对变量的重新定义。显示这一点的目的是平息你关于插入这种blatent效率低下的紧张局势:)。

> 其余的优化器运行后，我们得到：
```sh
define double @fib(double %x) {
entry:
  %cmptmp = fcmp ult double %x, 3.000000e+00
  %booltmp = uitofp i1 %cmptmp to double
  %ifcond = fcmp ueq double %booltmp, 0.000000e+00
  br i1 %ifcond, label %else, label %ifcont

else:
  %subtmp = fsub double %x, 1.000000e+00
  %calltmp = call double @fib(double %subtmp)
  %subtmp5 = fsub double %x, 2.000000e+00
  %calltmp6 = call double @fib(double %subtmp5)
  %addtmp = fadd double %calltmp, %calltmp6
  ret double %addtmp

ifcont:
  ret double 1.000000e+00
}
```
> 在这里我们看到，simplifycfg传递决定将返回指令克隆到'else'块的末尾。这允许它消除一些分支和PHI节点。

> 现在所有符号表引用都更新为使用堆栈变量，我们将添加赋值运算符。

## 7.6 新的赋值运算符
> 使用我们当前的框架，添加新的赋值运算符非常简单。我们将像任何其他二元运算符一样解析它，但在内部处理它（而不是允许用户定义它）。第一步是设置优先级：
```cpp
int main() {
  // Install standard binary operators.
  // 1 is lowest precedence.
  BinopPrecedence['='] = 2;
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
现在解析器知道二元运算符的优先级，它负责所有解析和AST生成。我们只需要为赋值运算符实现codegen。这看起来像：

Value *BinaryExprAST::codegen() {
  // Special case '=' because we don't want to emit the LHS as an expression.
  if (Op == '=') {
    // Assignment requires the LHS to be an identifier.
    VariableExprAST *LHSE = dynamic_cast<VariableExprAST*>(LHS.get());
    if (!LHSE)
      return LogErrorV("destination of '=' must be a variable");
与其余的二元运算符不同，我们的赋值运算符不遵循“发出LHS，发出RHS，执行计算”模型。因此，在处理其他二元运算符之前，它将作为特殊情况处理。另一个奇怪的是它需要LHS作为变量。“（x + 1）= expr”无效 - 只允许使用“x = expr”之类的东西。

  // Codegen the RHS.
  Value *Val = RHS->codegen();
  if (!Val)
    return nullptr;

  // Look up the name.
  Value *Variable = NamedValues[LHSE->getName()];
  if (!Variable)
    return LogErrorV("Unknown variable name");

  Builder.CreateStore(Val, Variable);
  return Val;
}
...
```
> 一旦我们得到了变量，codegen'的赋值很简单：我们发出赋值的RHS，创建一个商店，然后返回计算值。返回值允许链接赋值，例如“X =（Y = Z）”。

> 现在我们有了一个赋值运算符，我们可以改变循环变量和参数。例如，我们现在可以运行如下代码：
```sh
# Function to print a double.
extern printd(x);

# Define ':' for sequencing: as a low-precedence operator that ignores operands
# and just returns the RHS.
def binary : 1 (x y) y;

def test(x)
  printd(x) :
  x = 4 :
  printd(x);

test(123);
```
> 运行时，此示例打印“123”，然后打印“4”，表明我们实际上确实改变了值！好的，我们现在正式实现了我们的目标：在一般情况下，要实现这一目标需要SSA构建。但是，为了真正有用，我们希望能够定义我们自己的局部变量，让我们接下来添加它！

## 7.7 用户定义的局部变量
> 添加var / in就像我们对Kaleidoscope进行的任何其他扩展一样：我们扩展词法分析器，解析器，AST和代码生成器。添加新的'var / in'结构的第一步是扩展词法分析器。和以前一样，这非常简单，代码如下所示：
```cpp
enum Token {
  ...
  // var definition
  tok_var = -13
...
}
...
static int gettok() {
...
    if (IdentifierStr == "in")
      return tok_in;
    if (IdentifierStr == "binary")
      return tok_binary;
    if (IdentifierStr == "unary")
      return tok_unary;
    if (IdentifierStr == "var")
      return tok_var;
    return tok_identifier;
...
```
> 下一步是定义我们将构造的AST节点。对于var / in，它看起来像这样：
```cpp
/// VarExprAST - Expression class for var/in
class VarExprAST : public ExprAST {
  std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames;
  std::unique_ptr<ExprAST> Body;

public:
  VarExprAST(std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames,
             std::unique_ptr<ExprAST> Body)
    : VarNames(std::move(VarNames)), Body(std::move(Body)) {}

  Value *codegen() override;
};
```
> var / in允许一次定义名称列表，每个名称可以选择具有初始化值。因此，我们在VarNames向量中捕获此信息。另外，var / in有一个body，允许这个body访问var / in定义的变量。

> 有了这个，我们可以定义解析器片段。我们要做的第一件事是将它添加为主要表达式：
```cpp
/// primary
///   ::= identifierexpr
///   ::= numberexpr
///   ::= parenexpr
///   ::= ifexpr
///   ::= forexpr
///   ::= varexpr
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
  case tok_if:
    return ParseIfExpr();
  case tok_for:
    return ParseForExpr();
  case tok_var:
    return ParseVarExpr();
  }
}
```
> 接下来我们定义ParseVarExpr：
```cpp
/// varexpr ::= 'var' identifier ('=' expression)?
//                    (',' identifier ('=' expression)?)* 'in' expression
static std::unique_ptr<ExprAST> ParseVarExpr() {
  getNextToken();  // eat the var.

  std::vector<std::pair<std::string, std::unique_ptr<ExprAST>>> VarNames;

  // At least one variable name is required.
  if (CurTok != tok_identifier)
    return LogError("expected identifier after var");
```
> 此代码的第一部分将标识符/ expr对列表解析为本地VarNames向量。
```cpp
while (1) {
  std::string Name = IdentifierStr;
  getNextToken();  // eat identifier.

  // Read the optional initializer.
  std::unique_ptr<ExprAST> Init;
  if (CurTok == '=') {
    getNextToken(); // eat the '='.

    Init = ParseExpression();
    if (!Init) return nullptr;
  }

  VarNames.push_back(std::make_pair(Name, std::move(Init)));

  // End of var list, exit loop.
  if (CurTok != ',') break;
  getNextToken(); // eat the ','.

  if (CurTok != tok_identifier)
    return LogError("expected identifier list after var");
}
```
> 解析完所有变量后，我们将解析主体并创建AST节点：
```cpp
  // At this point, we have to have 'in'.
  if (CurTok != tok_in)
    return LogError("expected 'in' keyword after 'var'");
  getNextToken();  // eat 'in'.

  auto Body = ParseExpression();
  if (!Body)
    return nullptr;

  return llvm::make_unique<VarExprAST>(std::move(VarNames),
                                       std::move(Body));
}
```
> 现在我们可以解析并表示代码，我们需要支持LLVM IR的发射。此代码从以下开始：
```cpp
Value *VarExprAST::codegen() {
  std::vector<AllocaInst *> OldBindings;

  Function *TheFunction = Builder.GetInsertBlock()->getParent();

  // Register all variables and emit their initializer.
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i) {
    const std::string &VarName = VarNames[i].first;
    ExprAST *Init = VarNames[i].second.get();
```
> 基本上它遍历所有变量，一次安装一个变量。对于我们放入符号表的每个变量，我们记住了我们在OldBindings中替换的先前值。
```cpp
  // Emit the initializer before adding the variable to scope, this prevents
  // the initializer from referencing the variable itself, and permits stuff
  // like this:
  //  var a = 1 in
  //    var a = a in ...   # refers to outer 'a'.
  Value *InitVal;
  if (Init) {
    InitVal = Init->codegen();
    if (!InitVal)
      return nullptr;
  } else { // If not specified, use 0.0.
    InitVal = ConstantFP::get(TheContext, APFloat(0.0));
  }

  AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, VarName);
  Builder.CreateStore(InitVal, Alloca);

  // Remember the old variable binding so that we can restore the binding when
  // we unrecurse.
  OldBindings.push_back(NamedValues[VarName]);

  // Remember this binding.
  NamedValues[VarName] = Alloca;
}
```
> 这里有更多的评论而不是代码。基本思想是我们发出初始化器，创建alloca，然后更新符号表以指向它。一旦所有变量都安装在符号表中，我们就会评估var / in表达式的主体：
```cpp
// Codegen the body, now that all vars are in scope.
Value *BodyVal = Body->codegen();
if (!BodyVal)
  return nullptr;
最后，在返回之前，我们恢复以前的变量绑定：

  // Pop all our variables from scope.
  for (unsigned i = 0, e = VarNames.size(); i != e; ++i)
    NamedValues[VarNames[i].first] = OldBindings[i];

  // Return the body computation.
  return BodyVal;
}
```
> 所有这一切的最终结果是我们得到了适当的范围变量定义，我们甚至（通常）允许它们的变异:)。

> 有了这个，我们完成了我们的目标。我们很好的迭代fib示例来自intro编译并运行得很好。mem2reg传递将我们所有的堆栈变量优化到SSA寄存器中，在需要的地方插入PHI节点，并且我们的前端仍然很简单：在任何地方都没有“迭代优势边界”计算。

## 7.8 完整的代码清单
> 以下是我们运行示例的完整代码清单，增强了可变变量和var / in支持。要构建此示例，请使用：
```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core mcjit native` -O3 -o toy
# Run
./toy
```
> 这是代码：[toy.cpp](./toy.cpp)

[下一页：编译为目标代码](../Chapter08/README.md)