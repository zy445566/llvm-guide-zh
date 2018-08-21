# 4.万花筒：添加JIT和优化器支持
* 第4章介绍
* 琐碎常数折叠
* LLVM优化通过
* 添加JIT编译器
* 完整的代码清单

## 4.1 第4章介绍
> 欢迎阅读“ 使用LLVM实现语言 ”教程的第4章。第1-3章描述了简单语言的实现，并增加了对生成LLVM IR的支持。本章介绍两种新技术：为您的语言添加优化器支持，以及添加JIT编译器支持。这些新增内容将演示如何为Kaleidoscope语言提供优质，高效的代码。

## 4.2 琐碎常数折叠
> 我们对第3章的演示优雅且易于扩展。不幸的是，它不会产生很棒的代码。但是，在编译简单代码时，IRBuilder确实给了我们明显的优化：
```sh
ready> def test(x) 1+2+x;
Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 3.000000e+00, %x
        ret double %addtmp
}
```
> 此代码不是通过解析输入构建的AST的文字转录。那将是：
```sh
ready> def test(x) 1+2+x;
Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 2.000000e+00, 1.000000e+00
        %addtmp1 = fadd double %addtmp, %x
        ret double %addtmp1
}
```
> 如上所述，常量折叠是一种非常常见且非常重要的优化：以至于许多语言实现者在其AST表示中实现常量折叠支持。

> 使用LLVM，您不需要AST中的此支持。由于构建LLVM IR的所有调用都通过LLVM IR构建器，因此构建器本身会在调用它时检查是否存在常量折叠机会。如果是这样，它只是执行常量折叠并返回常量而不是创建指令。

> 嗯，这很容易:)。实际上，我们建议IRBuilder在生成这样的代码时始终使用 。它的使用没有“语法开销”（你不必在任何地方使用常量检查来编译你的编译器）并且它可以大大减少在某些情况下生成的LLVM IR的数量（特别是对于具有宏预处理器的语言或使用很多常量）。

> 另一方面，IRBuilder它受到以下事实的限制：它在构建时与代码内联进行所有分析。如果你采取一个稍微复杂的例子：
```sh
ready> def test(x) (1+2+x)*(x+(1+2));
ready> Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double 3.000000e+00, %x
        %addtmp1 = fadd double %x, 3.000000e+00
        %multmp = fmul double %addtmp, %addtmp1
        ret double %multmp
}
```
> 在这种情况下，乘法的LHS和RHS是相同的值。我们真的很想看到它生成“ ”而不是计算“ ”两次。tmp = x+3; result = tmp*tmp;x+3

> 不幸的是，没有多少本地分析能够检测并纠正这一点。这需要两个转换：表达式的重新关联（使add的词法相同）和Common Subexpression Elimination（CSE）删除冗余的add指令。幸运的是，LLVM以“通过”的形式提供了您可以使用的各种优化。

## 4.3 LLVM优化通过
> LLVM提供了许多优化过程，它们可以执行许多不同类型的操作并具有不同的权衡。与其他系统不同，LLVM并不认为一组优化适用于所有语言和所有情况的错误概念。LLVM允许编译器实现者就要使用的优化，按什么顺序以及在什么情况下做出完整的决策。

> 作为一个具体的例子，LLVM支持“整个模块”传递，它们可以查看尽可能大的代码体（通常是整个文件，但如果在链接时运行，这可能是整个程序的重要部分） 。它还支持并包含“每个功能”通道，它一次只能在一个功能上运行，而不需要查看其他功能。有关传递及其运行方式的更多信息，请参阅 如何编写传递文档和 LLVM传递列表。

> 对于Kaleidoscope，我们目前正在生成函数，一次一个，当用户输入它们时。我们不是在这个设置中拍摄最终的优化体验，但我们也想要抓住简单快捷的东西可能。因此，当用户键入函数时，我们将选择运行一些按功能优化。如果我们想要创建一个“静态Kaleidoscope编译器”，我们将使用我们现在拥有的代码，除非我们推迟运行优化器直到整个文件被解析。

> 为了实现每个函数的优化，我们需要设置一个 FunctionPassManager来保存和组织我们想要运行的LLVM优化。完成后，我们可以添加一组优化来运行。对于我们想要优化的每个模块，我们需要一个新的FunctionPassManager，因此我们将编写一个函数来为我们创建和初始化模块和传递管理器：
```cpp
void InitializeModuleAndPassManager(void) {
  // Open a new module.
  TheModule = llvm::make_unique<Module>("my cool jit", TheContext);

  // Create a new pass manager attached to it.
  TheFPM = llvm::make_unique<FunctionPassManager>(TheModule.get());

  // Do simple "peephole" optimizations and bit-twiddling optzns.
  TheFPM->add(createInstructionCombiningPass());
  // Reassociate expressions.
  TheFPM->add(createReassociatePass());
  // Eliminate Common SubExpressions.
  TheFPM->add(createGVNPass());
  // Simplify the control flow graph (deleting unreachable blocks, etc).
  TheFPM->add(createCFGSimplificationPass());

  TheFPM->doInitialization();
}
```
> 此代码初始化全局模块TheModule，以及TheFPM附加到的函数传递管理器TheModule。一旦设置了传递管理器，我们就会使用一系列“添加”调用来添加一堆LLVM传递。

> 在这种情况下，我们选择添加四个优化过程。我们在这里选择的通道是一组非常标准的“清理”优化，可用于各种代码。我不会深入研究他们的所作所为，但相信我，他们是一个很好的起点:)。

> 一旦设置了PassManager，我们就需要使用它。我们通过在构造（in FunctionAST::codegen()）我们新创建的函数之后运行它来执行此操作 ，但在它返回到客户端之前：
```cpp
if (Value *RetVal = Body->codegen()) {
  // Finish off the function.
  Builder.CreateRet(RetVal);

  // Validate the generated code, checking for consistency.
  verifyFunction(*TheFunction);

  // Optimize the function.
  TheFPM->run(*TheFunction);

  return TheFunction;
}
```
> 如您所见，这非常简单。的 FunctionPassManager优化和更新，以代替LLVM功能*，提高了（希望）它的身体。有了这个，我们可以再次尝试我们的测试：
```sh
ready> def test(x) (1+2+x)*(x+(1+2));
ready> Read function definition:
define double @test(double %x) {
entry:
        %addtmp = fadd double %x, 3.000000e+00
        %multmp = fmul double %addtmp, %addtmp
        ret double %multmp
}
```
> 正如预期的那样，我们现在可以获得优化的代码，从每次执行此函数时都可以保存浮点加法指令。

> LLVM提供了各种可在某些情况下使用的优化。有关各种通行证的一些文档可用，但它不是很完整。另一个好的想法来源可以来自查看Clang开始运行的传球 。“ opt”工具允许您尝试从命令行传递，以便您可以查看它们是否执行任何操作。

> 现在我们已经从前端获得了合理的代码，让我们谈谈执行它！

## 4.4 添加JIT编译器
> LLVM IR中提供的代码可以应用各种各样的工具。例如，您可以对其运行优化（如上所述），您可以将其以文本或二进制形式转储，您可以将代码编译为某个目标的汇编文件（.s），或者您可以JIT编译它。LLVM IR表示的好处在于它是编译器的许多不同部分之间的“共同货币”。

> 在本节中，我们将向我们的解释器添加JIT编译器支持。我们想要Kaleidoscope的基本思想是让用户像现在一样输入函数体，但是立即评估他们输入的顶级表达式。例如，如果他们输入“1 + 2;”，我们应该评估如果他们定义了一个函数，他们应该可以从命令行调用它。

> 为此，我们首先准备环境以为当前本机目标创建代码并声明和初始化JIT。这是通过调用一些InitializeNativeTarget\*函数并添加一个全局变量TheJIT并在main以下位置初始化它来完成的 ：
```cpp
static std::unique_ptr<KaleidoscopeJIT> TheJIT;
...
int main() {
  InitializeNativeTarget();
  InitializeNativeTargetAsmPrinter();
  InitializeNativeTargetAsmParser();

  // Install standard binary operators.
  // 1 is lowest precedence.
  BinopPrecedence['<'] = 10;
  BinopPrecedence['+'] = 20;
  BinopPrecedence['-'] = 20;
  BinopPrecedence['*'] = 40; // highest.

  // Prime the first token.
  fprintf(stderr, "ready> ");
  getNextToken();

  TheJIT = llvm::make_unique<KaleidoscopeJIT>();

  // Run the main "interpreter loop" now.
  MainLoop();

  return 0;
}
```
> 我们还需要为JIT设置数据布局：
```cpp
void InitializeModuleAndPassManager(void) {
  // Open a new module.
  TheModule = llvm::make_unique<Module>("my cool jit", TheContext);
  TheModule->setDataLayout(TheJIT->getTargetMachine().createDataLayout());

  // Create a new pass manager attached to it.
  TheFPM = llvm::make_unique<FunctionPassManager>(TheModule.get());
  ...
```
> KaleidoscopeJIT类是一个专门为这些教程构建的简单JIT，可以在llvm-src / examples / Kaleidoscope / include / KaleidoscopeJIT.h的LLVM源代码中找到。在后面的章节中，我们将看看它是如何工作的并使用新功能扩展它，但是现在我们将把它当作给定的。它的API非常简单： addModule将一个LLVM IR模块添加到JIT，使其功能可用于执行; removeModule删除模块，释放与该模块中的代码关联的任何内存; 并findSymbol允许我们查找编译代码的指针。

> 我们可以使用这个简单的API并更改我们解析顶级表达式的代码，如下所示：
```cpp
static void HandleTopLevelExpression() {
  // Evaluate a top-level expression into an anonymous function.
  if (auto FnAST = ParseTopLevelExpr()) {
    if (FnAST->codegen()) {

      // JIT the module containing the anonymous expression, keeping a handle so
      // we can free it later.
      auto H = TheJIT->addModule(std::move(TheModule));
      InitializeModuleAndPassManager();

      // Search the JIT for the __anon_expr symbol.
      auto ExprSymbol = TheJIT->findSymbol("__anon_expr");
      assert(ExprSymbol && "Function not found");

      // Get the symbol's address and cast it to the right type (takes no
      // arguments, returns a double) so we can call it as a native function.
      double (*FP)() = (double (*)())(intptr_t)ExprSymbol.getAddress();
      fprintf(stderr, "Evaluated to %f\n", FP());

      // Delete the anonymous expression module from the JIT.
      TheJIT->removeModule(H);
    }
```
> 如果解析和codegen成功，则下一步是将包含顶级表达式的模块添加到JIT。我们通过调用addModule来执行此操作，addModule触发模块中所有函数的代码生成，并返回一个句柄，可用于稍后从JIT中删除模块。一旦模块被添加到JIT，它就不能再被修改，所以我们也打开一个新模块来通过调用来保存后续代码InitializeModuleAndPassManager()。

> 一旦我们将模块添加到JIT，我们需要获得指向最终生成代码的指针。我们通过调用JIT的findSymbol方法并传递顶级表达式函数的名称来完成此操作：__anon_expr。由于我们刚刚添加了这个函数，我们断言findSymbol返回了一个结果。

> 接下来，我们__anon_expr通过调用getAddress()符号来获取函数 的内存中地址。回想一下，我们将顶级表达式编译为一个自包含的LLVM函数，该函数不带参数并返回计算的double。因为LLVM JIT编译器与本机平台ABI匹配，这意味着您可以将结果指针强制转换为该类型的函数指针并直接调用它。这意味着，JIT编译代码与静态链接到应用程序的本机机器代码之间没有区别。

> 最后，由于我们不支持对顶级表达式进行重新评估，因此当我们完成释放相关内存时，我们会从JIT中删除该模块。但是，回想一下，我们之前创建了几行（via InitializeModuleAndPassManager）的模块 仍处于打开状态，等待添加新代码。

> 只需这两个变化，让我们看看万花筒现在如何运作！
```sh
ready> 4+5;
Read top-level expression:
define double @0() {
entry:
  ret double 9.000000e+00
}

Evaluated to 9.000000
```
> 好吧，这看起来基本上是有效的。该函数的转储显示了“无参数函数总是返回double”，我们为每个键入的顶级表达式合成。这演示了非常基本的功能，但我们可以做更多吗？
```sh
ready> def testfunc(x y) x + y*2;
Read function definition:
define double @testfunc(double %x, double %y) {
entry:
  %multmp = fmul double %y, 2.000000e+00
  %addtmp = fadd double %multmp, %x
  ret double %addtmp
}

ready> testfunc(4, 10);
Read top-level expression:
define double @1() {
entry:
  %calltmp = call double @testfunc(double 4.000000e+00, double 1.000000e+01)
  ret double %calltmp
}

Evaluated to 24.000000

ready> testfunc(5, 10);
ready> LLVM ERROR: Program used external function 'testfunc' which could not be resolved!
```
> 函数定义和调用也有效，但最后一行出了问题。电话看起来有效，发生了什么？正如您可能从API中猜到的那样，模块是JIT的分配单元，而testfunc是包含匿名表达式的同一模块的一部分。当我们从JIT中删除该模块以释放匿名表达式的内存时，我们删除了testfunc它的定义。然后，当我们第二次尝试调用testfunc时，JIT无法再找到它。

> 解决此问题的最简单方法是将匿名表达式放在与其余函数定义不同的模块中。只要每个被调用的函数都有一个原型，JIT就会愉快地解决跨模块边界的函数调用，并在调用之前添加到JIT中。通过将匿名表达式放在不同的模块中，我们可以删除它而不影响其余的功能。

> 事实上，我们将更进一步，将每个功能都放在自己的模块中。这样做可以让我们利用KaleidoscopeJIT的有用属性，使我们的环境更像REPL：函数可以多次添加到JIT（与每个函数必须具有唯一定义的模块不同）。当您在KaleidoscopeJIT中查找符号时，它将始终返回最新的定义：
```sh
ready> def foo(x) x + 1;
Read function definition:
define double @foo(double %x) {
entry:
  %addtmp = fadd double %x, 1.000000e+00
  ret double %addtmp
}

ready> foo(2);
Evaluated to 3.000000

ready> def foo(x) x + 2;
define double @foo(double %x) {
entry:
  %addtmp = fadd double %x, 2.000000e+00
  ret double %addtmp
}

ready> foo(2);
Evaluated to 4.000000
```
> 为了让每个函数都存在于自己的模块中，我们需要一种方法来重新生成我们打开的每个新模块中的先前函数声明：
```cpp
static std::unique_ptr<KaleidoscopeJIT> TheJIT;

...

Function *getFunction(std::string Name) {
  // First, see if the function has already been added to the current module.
  if (auto *F = TheModule->getFunction(Name))
    return F;

  // If not, check whether we can codegen the declaration from some existing
  // prototype.
  auto FI = FunctionProtos.find(Name);
  if (FI != FunctionProtos.end())
    return FI->second->codegen();

  // If no existing prototype exists, return null.
  return nullptr;
}

...

Value *CallExprAST::codegen() {
  // Look up the name in the global module table.
  Function *CalleeF = getFunction(Callee);

...

Function *FunctionAST::codegen() {
  // Transfer ownership of the prototype to the FunctionProtos map, but keep a
  // reference to it for use below.
  auto &P = *Proto;
  FunctionProtos[Proto->getName()] = std::move(Proto);
  Function *TheFunction = getFunction(P.getName());
  if (!TheFunction)
    return nullptr;
```
> 为了实现这一点，我们首先添加一个新的全局FunctionProtos，它包含每个函数的最新原型。我们还将添加一个方便的方法getFunction()来替换调用TheModule->getFunction()。我们的便捷方法搜索TheModule现有的函数声明，如果没有找到，则回退到从FunctionProtos生成新的声明。在CallExprAST::codegen()我们只需要更换调用TheModule->getFunction()。在FunctionAST::codegen()我们需要先更新FunctionProtos地图，然后调用getFunction()。完成此操作后，我们总是可以在当前模块中获取任何先前声明的函数的函数声明。

> 我们还需要更新HandleDefinition和HandleExtern：
```cpp
static void HandleDefinition() {
  if (auto FnAST = ParseDefinition()) {
    if (auto *FnIR = FnAST->codegen()) {
      fprintf(stderr, "Read function definition:");
      FnIR->print(errs());
      fprintf(stderr, "\n");
      TheJIT->addModule(std::move(TheModule));
      InitializeModuleAndPassManager();
    }
  } else {
    // Skip token for error recovery.
     getNextToken();
  }
}

static void HandleExtern() {
  if (auto ProtoAST = ParseExtern()) {
    if (auto *FnIR = ProtoAST->codegen()) {
      fprintf(stderr, "Read extern: ");
      FnIR->print(errs());
      fprintf(stderr, "\n");
      FunctionProtos[ProtoAST->getName()] = std::move(ProtoAST);
    }
  } else {
    // Skip token for error recovery.
    getNextToken();
  }
}
```
> 在HandleDefinition中，我们添加两行来将新定义的函数传递给JIT并打开一个新模块。在HandleExtern中，我们只需添加一行即可将原型添加到FunctionProtos中。

> 通过这些更改，让我们再次尝试我们的REPL（这次我删除了匿名函数的转储，你现在应该得到这个想法:)：
```sh
ready> def foo(x) x + 1;
ready> foo(2);
Evaluated to 3.000000

ready> def foo(x) x + 2;
ready> foo(2);
Evaluated to 4.000000
```
> 有用！

> 即使使用这个简单的代码，我们也会获得一些令人惊讶的强大功能 - 请查看：
```sh
ready> extern sin(x);
Read extern:
declare double @sin(double)

ready> extern cos(x);
Read extern:
declare double @cos(double)

ready> sin(1.0);
Read top-level expression:
define double @2() {
entry:
  ret double 0x3FEAED548F090CEE
}

Evaluated to 0.841471

ready> def foo(x) sin(x)*sin(x) + cos(x)*cos(x);
Read function definition:
define double @foo(double %x) {
entry:
  %calltmp = call double @sin(double %x)
  %multmp = fmul double %calltmp, %calltmp
  %calltmp2 = call double @cos(double %x)
  %multmp4 = fmul double %calltmp2, %calltmp2
  %addtmp = fadd double %multmp, %multmp4
  ret double %addtmp
}

ready> foo(4.0);
Read top-level expression:
define double @3() {
entry:
  %calltmp = call double @foo(double 4.000000e+00)
  ret double %calltmp
}

Evaluated to 1.000000
```
> 哇，JIT怎么知道sin和cos？答案非常简单：KaleidoscopeJIT有一个简单的符号解析规则，用于查找任何给定模块中不可用的符号：首先，它搜索已经添加到JIT的所有模块，从最近的到最古老的，找到最新的定义。如果在JIT中没有找到定义，它将回退到dlsym("sin")Kaleidoscope进程本身上调用“ ”。由于“ sin”是在JIT的地址空间中定义的，它只是修补模块中的调用以调用libm版本sin直。但在某些情况下，这甚至更进一步：因为sin和cos是标准数学函数的名称，常量文件夹将在使用sin(1.0)上面“常量”中的常量调用时直接评估函数调用正确结果。

> 在未来，我们将看到如何调整此符号解析规则，以启用各种有用的功能，从安全性（限制JIT代码可用的符号集）到基于符号名称的动态代码生成，以及甚至是懒惰的编译。

> 符号解析规则的一个直接好处是我们现在可以通过编写任意C ++代码来实现操作来扩展语言。例如，如果我们添加：
```sh
#ifdef _WIN32
#define DLLEXPORT __declspec(dllexport)
#else
#define DLLEXPORT
#endif

/// putchard - putchar that takes a double and returns 0.
extern "C" DLLEXPORT double putchard(double X) {
  fputc((char)X, stderr);
  return 0;
}
```
> 注意，对于Windows，我们需要实际导出函数，因为动态符号加载器将使用GetProcAddress来查找符号。

> 现在，我们可以使用以下内容生成简单的输出到控制台：“ ”，在控制台上打印小写的“x”（120是“x”的ASCII代码）。类似的代码可用于在Kaleidoscope中实现文件I / O，控制台输入和许多其他功能。extern putchard(x); putchard(120);

> 这完成了Kaleidoscope教程的JIT和优化器章节。此时，我们可以编译非Turing完整的编程语言，优化和JIT以用户驱动的方式编译它。接下来我们将研究用控制流构造扩展语言，解决一些有趣的LLVM IR问题。

## 4.5 完整的代码清单
> 以下是我们的运行示例的完整代码清单，使用LLVM JIT和优化器进行了增强。要构建此示例，请使用：
```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core mcjit native` -O3 -o toy
# Run
./toy
```
> 如果您在Linux上编译它，请确保添加“-rdynamic”选项。这可确保在运行时正确解析外部函数。

> 这是代码：[toy.cpp](./toy.cpp)

[下一页：扩展语言：控制流程](../Chapter5/README.md)