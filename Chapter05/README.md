# 5.万花筒：扩展语言：控制流程
* 第5章简介
* if/then/ ELSE
* 用于If / Then / Else的Lexer扩展
* If / Then / Else的AST扩​​展
* If / Then / Else的解析扩展
* 用于If / Then / Else的LLVM IR
* If / Then / Else的代码生成
* 'for'Loop Expression
* “for”循环的Lexer扩展
* 'for'循环的AST扩​​展
* 'for'循环的解析器扩展
* 用于'for'循环的LLVM IR
* 'for'循环的代码生成
* 完整的代码清单
## 5.1 第5章介绍
> 欢迎阅读“ 使用LLVM实现语言 ”教程的第5章。第1-4部分介绍了简单的Kaleidoscope语言的实现，并包括对生成LLVM IR的支持，然后是优化和JIT编译器。不幸的是，正如所呈现的那样，万花筒几乎没用：它除了呼叫和返回之外没有其他控制流。这意味着您不能在代码中使用条件分支，从而显着限制其功能。在本期“构建编译器”中，我们将扩展Kaleidoscope，使其具有if / then / else表达式和一个简单的'for'循环。

## 5.2 if/ Then / 
> 扩展Kaleidoscope以支持if / then / else非常简单。它基本上需要为词法分析器，解析器，AST和LLVM代码发射器添加对这个“新”概念的支持。这个例子很好，因为它显示了随着时间的推移“增长”一种语言是多么容易，随着新想法的发现逐渐扩展它。

> 在我们开始“如何”添加此扩展之前，让我们谈谈我们想要的“什么”。基本的想法是我们希望能够写出这样的东西：
```sh
def fib(x)
  if x < 3 then
    1
  else
    fib(x-1)+fib(x-2);
```
> 在Kaleidoscope中，每个构造都是一个表达式：没有语句。因此，if / then / else表达式需要返回与其他任何值相同的值。由于我们使用的是大部分功能形式，我们将对其进行评估，然后根据条件的解析方式返回“then”或“else”值。这与C“？：”表达非常相似。

> if / then / else表达式的语义是它将条件计算为布尔相等值：0.0被认为是假，其他一切都被认为是真。如果条件为真，则计算并返回第一个子表达式，如果条件为假，则计算并返回第二个子表达式。由于万花筒允许副作用，因此这种行为对于确定是非常重要的。

> 现在我们知道了我们“想要”的东西，让我们把它分解成它的组成部分。

## 5.2.1 If / Then / Else的Lexer扩展
> 词法分析器扩展很简单。首先，我们为相关标记添加新的枚举值：
```sh
// control
tok_if = -6,
tok_then = -7,
tok_else = -8,
```
> 一旦我们有了这个，我们就会识别词法分析器中的新关键字。这很简单：

```cpp
...
if (IdentifierStr == "def")
  return tok_def;
if (IdentifierStr == "extern")
  return tok_extern;
if (IdentifierStr == "if")
  return tok_if;
if (IdentifierStr == "then")
  return tok_then;
if (IdentifierStr == "else")
  return tok_else;
return tok_identifier;
```
## 5.2.2 If / Then / Else的AST扩​​展
> 为了表示新表达式，我们为它添加了一个新的AST节点：
```cpp
/// IfExprAST - Expression class for if/then/else.
class IfExprAST : public ExprAST {
  std::unique_ptr<ExprAST> Cond, Then, Else;

public:
  IfExprAST(std::unique_ptr<ExprAST> Cond, std::unique_ptr<ExprAST> Then,
            std::unique_ptr<ExprAST> Else)
    : Cond(std::move(Cond)), Then(std::move(Then)), Else(std::move(Else)) {}

  Value *codegen() override;
};
```
> AST节点只有指向各种子表达式的指针。

## 5.2.3 If / Then / Else的解析扩展
> 现在我们有来自词法分析器的相关标记，并且我们要构建AST节点，我们的解析逻辑相对简单。首先我们定义一个新的解析函数：
```cpp
/// ifexpr ::= 'if' expression 'then' expression 'else' expression
static std::unique_ptr<ExprAST> ParseIfExpr() {
  getNextToken();  // eat the if.

  // condition.
  auto Cond = ParseExpression();
  if (!Cond)
    return nullptr;

  if (CurTok != tok_then)
    return LogError("expected then");
  getNextToken();  // eat the then

  auto Then = ParseExpression();
  if (!Then)
    return nullptr;

  if (CurTok != tok_else)
    return LogError("expected else");

  getNextToken();

  auto Else = ParseExpression();
  if (!Else)
    return nullptr;

  return llvm::make_unique<IfExprAST>(std::move(Cond), std::move(Then),
                                      std::move(Else));
}
```
> 接下来我们将它作为主要表达式连接起来：
```cpp
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
  }
}
```
## 5.2.4 用于If / Then / Else的
> 现在我们已经解析并构建了AST，最后一部分是添加LLVM代码生成支持。这是if / then / else示例中最有趣的部分，因为这是它开始引入新概念的地方。上面的所有代码都已在前面的章节中详细介绍过。

> 为了激发我们想要生成的代码，让我们看一个简单的例子。考虑：
```sh
extern foo();
extern bar();
def baz(x) if x then foo() else bar();
```
> 如果您禁用优化，您（很快）从Kaleidoscope获取的代码如下所示：
```sh
declare double @foo()

declare double @bar()

define double @baz(double %x) {
entry:
  %ifcond = fcmp one double %x, 0.000000e+00
  br i1 %ifcond, label %then, label %else

then:       ; preds = %entry
  %calltmp = call double @foo()
  br label %ifcont

else:       ; preds = %entry
  %calltmp1 = call double @bar()
  br label %ifcont

ifcont:     ; preds = %else, %then
  %iftmp = phi double [ %calltmp, %then ], [ %calltmp1, %else ]
  ret double %iftmp
}
```
> 要可视化控制流图，您可以使用LLVM“ opt ”工具的一个漂亮功能。如果你把这个LLVM IR放到“t.ll”并运行“ ”，会弹出一个窗口，你会看到这个图：llvm-as < t.ll | opt -analyze -view-cfg

![LangImpl05-cfg.png](./LangImpl05-cfg.png)

> 另一种方法是通过在代码中插入实际调用并重新编译或在调试器中调用它们来调用“ F->viewCFG()”或“ F->viewCFGOnly()”（其中F是“ Function*”）。LLVM具有许多用于可视化各种图形的很好的功能。

> 回到生成的代码，它非常简单：条目块评估条件表达式（在我们的例子中为“x”），并将结果与​​“ ”指令“（”一个“是”有序和不等于“）进行比较。 ）。根据此表达式的结果，代码跳转到“then”或“else”块，其中包含true / false情况的表达式。fcmp one

> 一旦then / else块完成执行，它们都会回到'ifcont'块以执行在if / then / else之后发生的代码。在这种情况下，唯一要做的就是返回函数的调用者。那么问题就变成了：代码如何知道返回哪个表达式？

> 这个问题的答案涉及一个重要的SSA操作： Phi操作。如果你不熟悉SSA，维基百科的文章 是一个很好的介绍，你最喜欢的搜索引擎上有各种其他介绍。简短的版本是Phi操作的“执行”需要“记住”哪个块控制来自。Phi操作采用与输入控制块相对应的值。在这种情况下，如果控制来自“then”块，它将获得“calltmp”的值。如果控制来自“else”块，则它获得“calltmp1”的值。

> 在这一点上，你可能开始想“哦不！这意味着我的简单优雅的前端必须开始生成SSA表单才能使用LLVM！“ 幸运的是，事实并非如此，我们强烈建议不要在前端实施SSA构造算法，除非有一个非常好的理由这样做。实际上，在为可能需要Phi节点的普通命令式编程语言编写的代码中，有两种值可以浮动：

> 涉及用户变量的代码： x = 1; x = x + 1;
> AST结构中隐含的值，例如本例中的Phi节点。
> 在本教程的第7章（“可变变量”）中，我们将深入讨论＃1。现在，请相信我，你不需要SSA构造来处理这种情况。对于＃2，您可以选择使用我们将为＃1描述的技术，或者如果方便的话，您可以直接插入Phi节点。在这种情况下，生成Phi节点非常容易，因此我们选择直接进行。

> 好的，足够的动力和概述，让我们生成代码！

## 5.2.5 If / Then / Else的代码生成
> 为了生成代码，我们实现了以下codegen方法IfExprAST：
```cpp
Value *IfExprAST::codegen() {
  Value *CondV = Cond->codegen();
  if (!CondV)
    return nullptr;

  // Convert condition to a bool by comparing non-equal to 0.0.
  CondV = Builder.CreateFCmpONE(
      CondV, ConstantFP::get(TheContext, APFloat(0.0)), "ifcond");
```
> 这段代码很简单，与我们之前看到的类似。我们为条件发出表达式，然后将该值与零进行比较，以获得真值作为1位（bool）值。
```cpp
Function *TheFunction = Builder.GetInsertBlock()->getParent();

// Create blocks for the then and else cases.  Insert the 'then' block at the
// end of the function.
BasicBlock *ThenBB =
    BasicBlock::Create(TheContext, "then", TheFunction);
BasicBlock *ElseBB = BasicBlock::Create(TheContext, "else");
BasicBlock *MergeBB = BasicBlock::Create(TheContext, "ifcont");

Builder.CreateCondBr(CondV, ThenBB, ElseBB);
```
> 此代码创建与if / then / else语句相关的基本块，并直接对应于上例中的块。第一行获取正在构建的当前Function对象。它通过询问构建器当前的BasicBlock，并询问该块的“父”（它当前嵌入的函数）来获得这一点。

> 一旦有了，就会创建三个块。请注意，它将“TheFunction”传递给“then”块的构造函数。这会导致构造函数自动将新块插入指定函数的末尾。创建了另外两个块，但尚未插入到该函数中。

> 一旦创建了块，我们就可以发出在它们之间选择的条件分支。请注意，创建新块不会隐式影响IRBuilder，因此它仍然会插入条件进入的块中。另请注意，它正在创建“then”块和“else”块的分支，即使“else”块尚未插入到函数中。这一切都很好：它是LLVM支持前向引用的标准方式。
```cpp
// Emit then value.
Builder.SetInsertPoint(ThenBB);

Value *ThenV = Then->codegen();
if (!ThenV)
  return nullptr;

Builder.CreateBr(MergeBB);
// Codegen of 'Then' can change the current block, update ThenBB for the PHI.
ThenBB = Builder.GetInsertBlock();
```
> 插入条件分支后，我们移动构建器以开始插入“then”块。严格地说，此调用将插入点移动到指定块的末尾。但是，由于“then”块是空的，它也可以通过在块的开头插入来开始。:)

> 一旦设置了插入点，我们递归地编码来自AST的“then”表达式。为了完成“then”块，我们为合并块创建了一个无条件分支。LLVM IR的一个有趣（也是非常重要）方面是它需要使用控制流指令（例如return或branch）“终止”所有基本块。这意味着必须在LLVM IR中明确指出所有控制流，包括漏降。如果违反此规则，验证程序将发出错误。

> 这里的最后一行非常微妙，但非常重要。基本问题是当我们在合并块中创建Phi节点时，我们需要设置块/值对，以指示Phi将如何工作。重要的是，Phi节点期望在CFG中具有块的每个前任的条目。那么，为什么我们在将它们设置为ThenBB 5行以上时获取当前块？问题是“Then”表达式实际上可能会改变Builder发出的块，例如，如果它包含嵌套的“if / then / else”表达式。因为codegen()递归调用可以任意改变当前块的概念，所以我们需要获得将设置Phi节点的代码的最新值。
```cpp
// Emit else block.
TheFunction->getBasicBlockList().push_back(ElseBB);
Builder.SetInsertPoint(ElseBB);

Value *ElseV = Else->codegen();
if (!ElseV)
  return nullptr;

Builder.CreateBr(MergeBB);
// codegen of 'Else' can change the current block, update ElseBB for the PHI.
ElseBB = Builder.GetInsertBlock();
```
> 'else'块的代码生成与'then'块的codegen基本相同。唯一显着的区别是第一行，它将'else'块添加到函数中。先前回想一下，'else'块已创建，但未添加到该函数中。既然发出了'then'和'else'块，我们可以使用合并代码完成：
```cpp
  // Emit merge block.
  TheFunction->getBasicBlockList().push_back(MergeBB);
  Builder.SetInsertPoint(MergeBB);
  PHINode *PN =
    Builder.CreatePHI(Type::getDoubleTy(TheContext), 2, "iftmp");

  PN->addIncoming(ThenV, ThenBB);
  PN->addIncoming(ElseV, ElseBB);
  return PN;
}
```
> 这里的前两行现在很熟悉：第一行将“merge”块添加到Function对象（它之前是浮动的，就像上面的else块一样）。第二个更改插入点，以便新创建的代码将进入“合并”块。完成后，我们需要创建PHI节点并为PHI设置块/值对。

> 最后，CodeGen函数返回phi节点作为if / then / else表达式计算的值。在上面的示例中，此返回值将提供给顶级函数的代码，该函数将创建返回指令。

> 总的来说，我们现在能够在Kaleidoscope中执行条件代码。通过此扩展，Kaleidoscope是一种相当完整的语言，可以计算各种数字函数。接下来我们将添加另一个非功能语言熟悉的有用表达式...

## 5.3 'for'循环表达式
> 现在我们知道如何向语言添加基本控制流构造，我们有了添加更多功能的工具。让我们添加更积极的东西，一个'for'表达式：
```sh
extern putchard(char);
def printstar(n)
  for i = 1, i < n, 1.0 in
    putchard(42);  # ascii 42 = '*'

# print 100 '*' characters
printstar(100);
```
> 该表达式定义了一个新变量（在这种情况下为“i”），它从起始值迭代，而条件（在这种情况下为“i <n”）为真，递增一个可选步长值（在这种情况下为“1.0”） ）。如果省略步长值，则默认为1.0。循环为true时，它会执行其正文表达式。因为我们没有更好的回报，所以我们只需将循环定义为始终返回0.0。将来当我们有可变变量时，它会变得更有用。

> 和以前一样，让我们​​来谈谈我们需要Kaleidoscope来支持这一变化。

## 5.3.1 'for'循环的Lexer扩展
> 词法分析器扩展与if / then / else相同：
```cpp
... in enum Token ...
// control
tok_if = -6, tok_then = -7, tok_else = -8,
tok_for = -9, tok_in = -10

... in gettok ...
if (IdentifierStr == "def")
  return tok_def;
if (IdentifierStr == "extern")
  return tok_extern;
if (IdentifierStr == "if")
  return tok_if;
if (IdentifierStr == "then")
  return tok_then;
if (IdentifierStr == "else")
  return tok_else;
if (IdentifierStr == "for")
  return tok_for;
if (IdentifierStr == "in")
  return tok_in;
return tok_identifier;
```
## 5.3.2 'for'循环的AST扩​​展
> AST节点也很简单。它基本上归结为捕获节点中的变量名称和组成表达式。
```cpp
/// ForExprAST - Expression class for for/in.
class ForExprAST : public ExprAST {
  std::string VarName;
  std::unique_ptr<ExprAST> Start, End, Step, Body;

public:
  ForExprAST(const std::string &VarName, std::unique_ptr<ExprAST> Start,
             std::unique_ptr<ExprAST> End, std::unique_ptr<ExprAST> Step,
             std::unique_ptr<ExprAST> Body)
    : VarName(VarName), Start(std::move(Start)), End(std::move(End)),
      Step(std::move(Step)), Body(std::move(Body)) {}

  Value *codegen() override;
};
```
## 5.3.3 'for'循环的解析器扩展
> 解析器代码也是相当标准的。这里唯一有趣的是处理可选的步骤值。解析器代码通过检查第二个逗号是否存在来处理它。如果不是，则在AST节点中将步长值设置为null：
```cpp
/// forexpr ::= 'for' identifier '=' expr ',' expr (',' expr)? 'in' expression
static std::unique_ptr<ExprAST> ParseForExpr() {
  getNextToken();  // eat the for.

  if (CurTok != tok_identifier)
    return LogError("expected identifier after for");

  std::string IdName = IdentifierStr;
  getNextToken();  // eat identifier.

  if (CurTok != '=')
    return LogError("expected '=' after for");
  getNextToken();  // eat '='.


  auto Start = ParseExpression();
  if (!Start)
    return nullptr;
  if (CurTok != ',')
    return LogError("expected ',' after for start value");
  getNextToken();

  auto End = ParseExpression();
  if (!End)
    return nullptr;

  // The step value is optional.
  std::unique_ptr<ExprAST> Step;
  if (CurTok == ',') {
    getNextToken();
    Step = ParseExpression();
    if (!Step)
      return nullptr;
  }

  if (CurTok != tok_in)
    return LogError("expected 'in' after for");
  getNextToken();  // eat 'in'.

  auto Body = ParseExpression();
  if (!Body)
    return nullptr;

  return llvm::make_unique<ForExprAST>(IdName, std::move(Start),
                                       std::move(End), std::move(Step),
                                       std::move(Body));
}
```
> 我们再次将它作为主要表达式连接起来：
```cpp
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
  }
}
```
## 5.3.4 用于'for'循环的
> 现在我们得到了很好的部分：我们想为这件事生成的LLVM IR。通过上面的简单示例，我们获得了这个LLVM IR（请注意，为了清晰起见，生成此转储并禁用优化）：
```sh
declare double @putchard(double)

define double @printstar(double %n) {
entry:
  ; initial value = 1.0 (inlined into phi)
  br label %loop

loop:       ; preds = %loop, %entry
  %i = phi double [ 1.000000e+00, %entry ], [ %nextvar, %loop ]
  ; body
  %calltmp = call double @putchard(double 4.200000e+01)
  ; increment
  %nextvar = fadd double %i, 1.000000e+00

  ; termination test
  %cmptmp = fcmp ult double %i, %n
  %booltmp = uitofp i1 %cmptmp to double
  %loopcond = fcmp one double %booltmp, 0.000000e+00
  br i1 %loopcond, label %loop, label %afterloop

afterloop:      ; preds = %loop
  ; loop always returns 0.0
  ret double 0.000000e+00
}
```
> 这个循环包含我们之前看到的所有相同结构：一个phi节点，几个表达式和一些基本块。让我们看看这是如何组合在一起的。

## 5.3.5 “for”循环的代码生成
> codegen的第一部分非常简单：我们只输出循环值的起始表达式：
```cpp
Value *ForExprAST::codegen() {
  // Emit the start code first, without 'variable' in scope.
  Value *StartVal = Start->codegen();
  if (!StartVal)
    return nullptr;
```
> 完成此操作后，下一步是为循环体的启动设置LLVM基本块。在上面的例子中，整个循环体是一个块，但请记住，体代码本身可以由多个块组成（例如，如果它包含if / then / else或for / in表达式）。
```cpp
// Make the new basic block for the loop header, inserting after current
// block.
Function *TheFunction = Builder.GetInsertBlock()->getParent();
BasicBlock *PreheaderBB = Builder.GetInsertBlock();
BasicBlock *LoopBB =
    BasicBlock::Create(TheContext, "loop", TheFunction);

// Insert an explicit fall through from the current block to the LoopBB.
Builder.CreateBr(LoopBB);
```
> 这段代码与我们在if / then / else中看到的类似。因为我们需要它来创建Phi节点，所以我们记住了进入循环的块。一旦我们有了这个，我们创建实际的块来启动循环并为两个块之间的连接创建一个无条件分支。
```cpp
// Start insertion in LoopBB.
Builder.SetInsertPoint(LoopBB);

// Start the PHI node with an entry for Start.
PHINode *Variable = Builder.CreatePHI(Type::getDoubleTy(TheContext),
                                      2, VarName.c_str());
Variable->addIncoming(StartVal, PreheaderBB);
```
> 现在已经设置了循环的“预读器”，我们切换到循环体的发射代码。首先，我们移动插入点并为循环归纳变量创建PHI节点。由于我们已经知道起始值的传入值，因此我们将其添加到Phi节点。请注意，Phi最终将获得备份的第二个值，但我们还无法设置它（因为它不存在！）。
```cpp
// Within the loop, the variable is defined equal to the PHI node.  If it
// shadows an existing variable, we have to restore it, so save it now.
Value *OldVal = NamedValues[VarName];
NamedValues[VarName] = Variable;

// Emit the body of the loop.  This, like any other expr, can change the
// current BB.  Note that we ignore the value computed by the body, but don't
// allow an error.
if (!Body->codegen())
  return nullptr;
```
> 现在代码开始变得更有趣了。我们的'for'循环为符号表引入了一个新变量。这意味着我们的符号表现在可以包含函数参数或循环变量。为了解决这个问题，在我们编写循环体之前，我们将循环变量添加为其名称的当前值。请注意，外部作用域中可能存在同名变量。很容易使这个错误（发出错误并返回null，如果已经存在VarName的条目）但我们选择允许变量的阴影。为了正确处理这个问题，我们记住了我们可能隐藏的值OldVal（如果没有阴影变量，则为null）。

> 一旦将循环变量设置到符号表中，代码就会递归代码为body。这允许正文使用循环变量：对它的任何引用自然会在符号表中找到它。
```cpp
// Emit the step value.
Value *StepVal = nullptr;
if (Step) {
  StepVal = Step->codegen();
  if (!StepVal)
    return nullptr;
} else {
  // If not specified, use 1.0.
  StepVal = ConstantFP::get(TheContext, APFloat(1.0));
}

Value *NextVar = Builder.CreateFAdd(Variable, StepVal, "nextvar");
```
> 现在身体被发射，我们通过添加步长值计算迭代变量的下一个值，如果不存在则计算1.0。' NextVar'将是循环的下一次迭代中循环变量的值。
```cpp
// Compute the end condition.
Value *EndCond = End->codegen();
if (!EndCond)
  return nullptr;

// Convert condition to a bool by comparing non-equal to 0.0.
EndCond = Builder.CreateFCmpONE(
    EndCond, ConstantFP::get(TheContext, APFloat(0.0)), "loopcond");
```
> 最后，我们评估循环的退出值，以确定循环是否应该退出。这反映了if / then / else语句的条件评估。
```cpp
// Create the "after loop" block and insert it.
BasicBlock *LoopEndBB = Builder.GetInsertBlock();
BasicBlock *AfterBB =
    BasicBlock::Create(TheContext, "afterloop", TheFunction);

// Insert the conditional branch into the end of LoopEndBB.
Builder.CreateCondBr(EndCond, LoopBB, AfterBB);

// Any new code will be inserted in AfterBB.
Builder.SetInsertPoint(AfterBB);
```
> 随着循环体的代码完成，我们只需要为它完成控制流程。此代码记住结束块（对于phi节点），然后为循环退出创建块（“afterloop”）。根据退出条件的值，它创建一个条件分支，在再次执行循环和退出循环之间选择。任何将来的代码都会在“afterloop”块中发出，因此它会为其设置插入位置。
```cpp
  // Add a new entry to the PHI node for the backedge.
  Variable->addIncoming(NextVar, LoopEndBB);

  // Restore the unshadowed variable.
  if (OldVal)
    NamedValues[VarName] = OldVal;
  else
    NamedValues.erase(VarName);

  // for expr always returns 0.0.
  return Constant::getNullValue(Type::getDoubleTy(TheContext));
}
```
> 最终代码处理各种清理：现在我们有“NextVar”值，我们可以将传入值添加到循环PHI节点。之后，我们从符号表中删除循环变量，使其在for循环后不在范围内。最后，for循环的代码生成总是返回0.0，这就是我们返回的内容 ForExprAST::codegen()。

> 有了这个，我们总结了本教程中的“向Kaleidoscope添加控制流”一章。在本章中，我们添加了两个控制流构造，并使用它们来激发LLVM IR的几个方面，这些方面对于前端实现者来说非常重要。在我们的传奇的下一章中，我们会变得有点疯狂，并为我们可怜的无辜语言添加用户定义的运算符。

## 5.4 完整的代码清单
> 以下是我们正在运行的示例的完整代码清单，使用if / then / else和for表达式进行了增强。要构建此示例，请使用：
```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core mcjit native` -O3 -o toy
# Run
./toy
```
> 这是代码：[toy.cpp](./toy.cpp)

[下一页：扩展语言：用户定义的运算符](../Chapter06/README.md)