# 9.万花筒：添加调试信息
* 第9章简介
* 为什么这是一个难题？
* 提前编译模式
* 编译单位
* DWARF排放设置
* 功能
* 来源地点
* 变量
* 完整的代码清单
## 9.1 第9章介绍
> 欢迎阅读“ 使用LLVM实现语言 ”教程的第9章。在第1章到第8章中，我们构建了一个包含函数和变量的小型编程语言。如果出现问题会发生什么，你如何调试你的程序？

> 源级调试使用格式化数据，帮助调试器从二进制文件和机器状态转换回程序员编写的源。在LLVM中，我们通常使用称为DWARF的格式。DWARF是一种紧凑的编码，表示类型，源位置和变量位置。

> 本章的简短摘要是，我们将介绍您必须添加到编程语言中以支持调试信息的各种内容，以及如何将其转换为DWARF。

> 警告：现在我们无法通过JIT进行调试，因此我们需要将程序编译成小而独立的程序。作为其中的一部分，我们将对语言的运行以及程序的编译方式进行一些修改。这意味着我们将拥有一个源文件，其中包含用Kaleidoscope编写的简单程序，而不是交互式JIT。它确实涉及一个限制，我们一次只能有一个“顶级”命令来减少必要的更改次数。

> 这是我们将要编译的示例程序：
```sh
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2);

fib(10)
```
## 9.2 为什么这是一个难题？
> 出于几个不同的原因，调试信息是一个难题 - 主要集中在优化代码上。首先，优化使得源位置更加困难。在LLVM IR中，我们保留指令上每个IR级指令的原始源位置。优化过程应保留新创建的指令的源位置，但合并的指令只能保留一个位置 - 这可能导致在逐步优化的程序时跳转。其次，优化可以以优化的方式移动变量，在内存中与其他变量共享或难以跟踪。出于本教程的目的，我们将避免优化（正如您将看到的下一组补丁之一）。

## 9.3 提前编译模式
> 为了突出显示将调试信息添加到源语言的方面，而不必担心JIT调试的复杂性，我们将对Kaleidoscope进行一些更改，以支持将前端发出的IR编译成一个简单的独立程序，您可以执行，调试和查看结果。

> 首先，我们将包含顶级语句的匿名函数作为我们的“主要”：
```cpp
-    auto Proto = llvm::make_unique<PrototypeAST>("", std::vector<std::string>());
+    auto Proto = llvm::make_unique<PrototypeAST>("main", std::vector<std::string>());
```
> 只需简单地改变它的名字即可。

> 然后我们将删除命令行代码，只要它存在：
```cpp
@@ -1129,7 +1129,6 @@ static void HandleTopLevelExpression() {
 /// top ::= definition | external | expression | ';'
 static void MainLoop() {
   while (1) {
-    fprintf(stderr, "ready> ");
     switch (CurTok) {
     case tok_eof:
       return;
@@ -1184,7 +1183,6 @@ int main() {
   BinopPrecedence['*'] = 40; // highest.

   // Prime the first token.
-  fprintf(stderr, "ready> ");
   getNextToken();
```
> 最后，我们将禁用所有优化传递和JIT，以便在我们完成解析和生成代码之后发生的唯一事情是LLVM IR转到标准错误：
```cpp
@@ -1108,17 +1108,8 @@ static void HandleExtern() {
 static void HandleTopLevelExpression() {
   // Evaluate a top-level expression into an anonymous function.
   if (auto FnAST = ParseTopLevelExpr()) {
-    if (auto *FnIR = FnAST->codegen()) {
-      // We're just doing this to make sure it executes.
-      TheExecutionEngine->finalizeObject();
-      // JIT the function, returning a function pointer.
-      void *FPtr = TheExecutionEngine->getPointerToFunction(FnIR);
-
-      // Cast it to the right type (takes no arguments, returns a double) so we
-      // can call it as a native function.
-      double (*FP)() = (double (*)())(intptr_t)FPtr;
-      // Ignore the return value for this.
-      (void)FP;
+    if (!F->codegen()) {
+      fprintf(stderr, "Error generating code for top level expr");
     }
   } else {
     // Skip token for error recovery.
@@ -1439,11 +1459,11 @@ int main() {
   // target lays out data structures.
   TheModule->setDataLayout(TheExecutionEngine->getDataLayout());
   OurFPM.add(new DataLayoutPass());
+#if 0
   OurFPM.add(createBasicAliasAnalysisPass());
   // Promote allocas to registers.
   OurFPM.add(createPromoteMemoryToRegisterPass());
@@ -1218,7 +1210,7 @@ int main() {
   OurFPM.add(createGVNPass());
   // Simplify the control flow graph (deleting unreachable blocks, etc).
   OurFPM.add(createCFGSimplificationPass());
-
+  #endif
   OurFPM.doInitialization();

   // Set the global so the code gen can use this.
```
> 这一相对较小的更改使我们能够通过此命令行将我们的Kaleidoscope语言编译为可执行程序：
```sh
Kaleidoscope-Ch9 < fib.ks | & clang -x ir -
```
> 它在当前工作目录中提供了a.out / a.exe。

## 9.4 编译单元
> DWARF中一段代码的顶级容器是一个编译单元。它包含单个翻译单元的类型和功能数据（读取：源代码的一个文件）。所以我们需要做的第一件事是为fib.ks文件构造一个。

## 9.5 DWARF发射设置
> 与IRBuilder类类似，我们有一个 DIBuilder类，可以帮助构建LLVM IR文件的调试元数据。它IRBuilder与LLVM IR 类似地对应1：1 ，但具有更好的名称。使用它确实需要您比您需要的更熟悉DWARF术语IRBuilder和Instruction名称，但如果您阅读元数据格式的一般文档， 它应该更清楚一点。我们将使用此类来构建所有IR级别描述。它的构造需要一个模块，所以我们需要在构建模块后不久构建它。我们把它作为一个全局静态变量，使它更容易使用。

> 接下来我们将创建一个小容器来缓存我们的一些频繁数据。第一个是我们的编译单元，但是我们也会为我们的一个类型编写一些代码，因为我们不必担心多个类型表达式：
```cpp
static DIBuilder *DBuilder;

struct DebugInfo {
  DICompileUnit *TheCU;
  DIType *DblTy;

  DIType *getDoubleTy();
} KSDbgInfo;

DIType *DebugInfo::getDoubleTy() {
  if (DblTy)
    return DblTy;

  DblTy = DBuilder->createBasicType("double", 64, dwarf::DW_ATE_float);
  return DblTy;
}
```
> 然后在main我们构建模块的时候：
```cpp
DBuilder = new DIBuilder(*TheModule);

KSDbgInfo.TheCU = DBuilder->createCompileUnit(
    dwarf::DW_LANG_C, DBuilder->createFile("fib.ks", "."),
    "Kaleidoscope Compiler", 0, "", 0);
```
> 这里有几点需要注意。首先，当我们为一个名为Kaleidoscope的语言生成一个编译单元时，我们使用C语言常量。这是因为调试器不一定理解它不能识别的语言的调用约定或默认ABI，我们遵循我们的LLVM代码生成中的C ABI因此它是最接近准确的。这确保了我们可以实际调用调试器中的函数并让它们执行。其次，你会在调用中看到“fib.ks” createCompileUnit。这是默认的硬编码值，因为我们使用shell重定向将我们的源代码放入Kaleidoscope编译器中。在通常的前端，你有一个输入文件名，它会去那里。

> 作为通过DIBuilder发出调试信息的一部分，最后一件事是我们需要“完成”调试信息。原因是DIBuilder底层API的一部分，但请确保在main结束时执行此操作：
```cpp
DBuilder->finalize();
```
> 在转储模块之前。

## 9.6 功能
> 现在我们有了源位置，我们可以在调试信息中添加函数定义。因此，我们添加几行代码来描述子程序的上下文，在本例中为“File”，以及函数本身的实际定义。Compile UnitPrototypeAST::codegen()

> 所以上下文：
```cpp
DIFile *Unit = DBuilder->createFile(KSDbgInfo.TheCU.getFilename(),
                                    KSDbgInfo.TheCU.getDirectory());
```
> 给我们一个DIFile并询问我们上面创建的目录和文件名。然后，现在，我们使用0的一些源位置（因为我们的AST当前没有源位置信息）并构造我们的函数定义：Compile Unit
```cpp
DIScope *FContext = Unit;
unsigned LineNo = 0;
unsigned ScopeLine = 0;
DISubprogram *SP = DBuilder->createFunction(
    FContext, P.getName(), StringRef(), Unit, LineNo,
    CreateFunctionType(TheFunction->arg_size(), Unit),
    false /* internal linkage */, true /* definition */, ScopeLine,
    DINode::FlagPrototyped, false);
TheFunction->setSubprogram(SP);
```
> 我们现在有一个DISubprogram，其中包含对该函数的所有元数据的引用。

## 9.7 源位置
> 调试信息最重要的是准确的源位置 - 这使得可以将源代码映射回来。我们遇到了一个问题，Kaleidoscope在词法分析器或解析器中确实没有任何源位置信息，所以我们需要添加它。
```cpp
struct SourceLocation {
  int Line;
  int Col;
};
static SourceLocation CurLoc;
static SourceLocation LexLoc = {1, 0};

static int advance() {
  int LastChar = getchar();

  if (LastChar == '\n' || LastChar == '\r') {
    LexLoc.Line++;
    LexLoc.Col = 0;
  } else
    LexLoc.Col++;
  return LastChar;
}
```
> 在这组代码中，我们添加了一些关于如何跟踪“源文件”的行和列的功能。当我们使用每个令牌时，我们将当前当前的“词汇位置”设置为令牌开头的各种行和列。我们通过覆盖所有先前调用 getchar()我们的新事件advance()来跟踪信息，然后我们将所有AST类添加到源位置：
```cpp
class ExprAST {
  SourceLocation Loc;

  public:
    ExprAST(SourceLocation Loc = CurLoc) : Loc(Loc) {}
    virtual ~ExprAST() {}
    virtual Value* codegen() = 0;
    int getLine() const { return Loc.Line; }
    int getCol() const { return Loc.Col; }
    virtual raw_ostream &dump(raw_ostream &out, int ind) {
      return out << ':' << getLine() << ':' << getCol() << '\n';
    }
```
> 当我们创建一个新表达式时，我们传递下去：
```cpp
LHS = llvm::make_unique<BinaryExprAST>(BinLoc, BinOp, std::move(LHS),
                                       std::move(RHS));
```
> 为我们提供每个表达式和变量的位置。

> 为了确保每条指令都能获得正确的源位置信息，我们必须告知Builder每当我们处于新的源位置时。我们使用一个小辅助函数：
```cpp
void DebugInfo::emitLocation(ExprAST *AST) {
  DIScope *Scope;
  if (LexicalBlocks.empty())
    Scope = TheCU;
  else
    Scope = LexicalBlocks.back();
  Builder.SetCurrentDebugLocation(
      DebugLoc::get(AST->getLine(), AST->getCol(), Scope));
}
```
> 这两个都告诉IRBuilder我们主要的位置，以及我们所处的范围。范围可以是编译单元级别，也可以是最近的封闭词汇块，如当前函数。为了表示这一点，我们创建了一堆范围：
```cpp
std::vector<DIScope *> LexicalBlocks;
```
> 当我们开始为每个函数生成代码时，将范围（函数）推送到堆栈的顶部：
```cpp
KSDbgInfo.LexicalBlocks.push_back(SP);
```
> 此外，我们可能不会忘记在函数代码生成结束时从作用域堆栈中弹出作用域：
```cpp
// Pop off the lexical block for the function since we added it
// unconditionally.
KSDbgInfo.LexicalBlocks.pop_back();
```
> 然后我们确保每次开始为新的AST对象生成代码时发出位置：
```cpp
KSDbgInfo.emitLocation(this);
```
## 9.8 变量
> 现在我们有了函数，我们需要能够打印出范围内的变量。让我们设置我们的函数参数，这样我们就可以获得不错的回溯，看看我们的函数是如何被调用的。它不是很多代码，我们通常在创建参数allocas时处理它FunctionAST::codegen。
```cpp
// Record the function arguments in the NamedValues map.
NamedValues.clear();
unsigned ArgIdx = 0;
for (auto &Arg : TheFunction->args()) {
  // Create an alloca for this variable.
  AllocaInst *Alloca = CreateEntryBlockAlloca(TheFunction, Arg.getName());

  // Create a debug descriptor for the variable.
  DILocalVariable *D = DBuilder->createParameterVariable(
      SP, Arg.getName(), ++ArgIdx, Unit, LineNo, KSDbgInfo.getDoubleTy(),
      true);

  DBuilder->insertDeclare(Alloca, D, DBuilder->createExpression(),
                          DebugLoc::get(LineNo, 0, SP),
                          Builder.GetInsertBlock());

  // Store the initial value into the alloca.
  Builder.CreateStore(&Arg, Alloca);

  // Add arguments to variable symbol table.
  NamedValues[Arg.getName()] = Alloca;
}
```
> 这里我们首先创建变量，给它scope（SP），名称，源位置，类型，因为它是一个参数，参数索引。接下来，我们创建一个lvm.dbg.declare调用，在IR级别指示我们在alloca中有一个变量（它给出了变量的起始位置），并在声明上设置了范围开头的源位置。

> 此时需要注意的一件有趣的事情是，各种调试器都会根据过去为它们生成代码和调试信息的方式进行假设。在这种情况下，我们需要做一些hack以避免为函数序言生成行信息，以便调试器知道在设置断点时跳过这些指令。所以在 FunctionAST::CodeGen我们添加更多行：
```cpp
// Unset the location for the prologue emission (leading instructions with no
// location in a function are considered part of the prologue and the debugger
// will run past them when breaking on a function)
KSDbgInfo.emitLocation(nullptr);
```
> 然后在我们实际开始为函数体生成代码时发出一个新位置：
```cpp
KSDbgInfo.emitLocation(Body.get());
```
> 有了这个，我们有足够的调试信息来设置函数中的断点，打印出参数变量和调用函数。只需几行简单的代码就行了！

## 9.9 完整的代码清单
> 以下是我们的运行示例的完整代码清单，增强了调试信息。要构建此示例，请使用：
```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core mcjit native` -O3 -o toy
# Run
./toy
```
> 这是代码：[toy.cpp](./toy.cpp)

[下一篇：结论和其他有用的LLVM花絮](../Chapter10/README.md)