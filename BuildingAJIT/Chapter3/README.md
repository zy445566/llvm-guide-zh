# 3.构建JIT：按函数惰性编译
* 第3章简介
* 懒惰编译
* 完整的代码清单

## 3.1 第3章介绍
> 欢迎阅读“在LLVM中构建基于ORC的JIT”教程的第3章。本章讨论惰性JITing，并向您展示如何通过从第2章添加JIT的ORC CompileOnDemand层来启用它。

## 3.2 延迟编译
> 当我们从第2章向KaleidoscopeJIT类添加一个模块时，它会立即由​​IRTransformLayer，IRCompileLayer和RTDyldObjectLinkingLayer为我们进行优化，编译和链接。这个方案使得模块可执行文件的所有工作都是预先完成的，很容易理解，其性能特征很容易理解。但是，如果要编译的代码量很大，它将导致非常高的启动时间，如果在运行时只调用少量编译函数，也可能会进行大量不必要的编译。真正的“即时”编译器应该允许我们推迟任何给定函数的编译，直到首次调用该函数为止，从而缩短启动时间并消除冗余工作。事实上，ORC API为我们提供了一个懒惰地编译LLVM IR的层： CompileOnDemandLayer。

> CompileOnDemandLayer类符合第2章中描述的图层接口，但其addModule方法的行为与我们目前看到的图层完全不同：它不是预先做任何工作，而是扫描正在添加的模块并安排每个函数。它们在第一次被调用时被编译。为此，CompileOnDemandLayer为它扫描的每个函数创建两个小实用程序：存根和编译回调。存根是一对函数指针（一旦编译了函数将指向函数的实现）和间接跳过指针。通过在程序的生命周期内修复间接跳转的地址，我们可以为函数提供一个永久的“有效地址”，即使函数的实现从未编译，也可以安全地用于间接和函数指针比较。编译不止一次（例如，由于在更高的优化级别重新编译函数）并更改地址。第二个实用程序，即编译回调，表示从程序到编译器的重新进入点，它将触发编译然后执行函数。通过初始化函数的存根以指向函数的编译回调，我们启用延迟编译：第一次尝试调用函数将跟随函数指针并触发编译回调。编译回调将编译该函数，更新存根的函数指针，然后执行该函数。在对函数的所有后续调用中，函数指针将指向已编译的函数，因此编译器不会产生进一步的开销。我们将在本教程的下一章中更详细地介绍这个过程，但是现在我们将相信CompileOnDemandLayer为我们设置所有存根和回调。我们需要做的就是将CompileOnDemandLayer添加到堆栈的顶部，我们将获得延迟编译的好处。我们只需要对源进行一些更改：编译回调将编译该函数，更新存根的函数指针，然后执行该函数。在对函数的所有后续调用中，函数指针将指向已编译的函数，因此编译器不会产生进一步的开销。我们将在本教程的下一章中更详细地介绍这个过程，但是现在我们将相信CompileOnDemandLayer为我们设置所有存根和回调。我们需要做的就是将CompileOnDemandLayer添加到堆栈的顶部，我们将获得延迟编译的好处。我们只需要对源进行一些更改：编译回调将编译该函数，更新存根的函数指针，然后执行该函数。在对函数的所有后续调用中，函数指针将指向已编译的函数，因此编译器不会产生进一步的开销。我们将在本教程的下一章中更详细地介绍这个过程，但是现在我们将相信CompileOnDemandLayer为我们设置所有存根和回调。我们需要做的就是将CompileOnDemandLayer添加到堆栈的顶部，我们将获得延迟编译的好处。我们只需要对源进行一些更改：所以编译器没有进一步的开销。我们将在本教程的下一章中更详细地介绍这个过程，但是现在我们将相信CompileOnDemandLayer为我们设置所有存根和回调。我们需要做的就是将CompileOnDemandLayer添加到堆栈的顶部，我们将获得延迟编译的好处。我们只需要对源进行一些更改：所以编译器没有进一步的开销。我们将在本教程的下一章中更详细地介绍这个过程，但是现在我们将相信CompileOnDemandLayer为我们设置所有存根和回调。我们需要做的就是将CompileOnDemandLayer添加到堆栈的顶部，我们将获得延迟编译的好处。我们只需要对源进行一些更改：
```cpp
...
#include "llvm/ExecutionEngine/SectionMemoryManager.h"
#include "llvm/ExecutionEngine/Orc/CompileOnDemandLayer.h"
#include "llvm/ExecutionEngine/Orc/CompileUtils.h"
...

...
class KaleidoscopeJIT {
private:
  std::unique_ptr<TargetMachine> TM;
  const DataLayout DL;
  RTDyldObjectLinkingLayer ObjectLayer;
  IRCompileLayer<decltype(ObjectLayer), SimpleCompiler> CompileLayer;

  using OptimizeFunction =
      std::function<std::shared_ptr<Module>(std::shared_ptr<Module>)>;

  IRTransformLayer<decltype(CompileLayer), OptimizeFunction> OptimizeLayer;

  std::unique_ptr<JITCompileCallbackManager> CompileCallbackManager;
  CompileOnDemandLayer<decltype(OptimizeLayer)> CODLayer;

public:
  using ModuleHandle = decltype(CODLayer)::ModuleHandleT;
首先，我们需要包含CompileOnDemandLayer.h头，然后向我们的类添加两个新成员：std :: unique_ptr <JITCompileCallbackManager>和CompileOnDemandLayer。CompileOnDemandLayer使用CompileCallbackManager成员来创建每个函数所需的编译回调。

KaleidoscopeJIT()
    : TM(EngineBuilder().selectTarget()), DL(TM->createDataLayout()),
      ObjectLayer([]() { return std::make_shared<SectionMemoryManager>(); }),
      CompileLayer(ObjectLayer, SimpleCompiler(*TM)),
      OptimizeLayer(CompileLayer,
                    [this](std::shared_ptr<Module> M) {
                      return optimizeModule(std::move(M));
                    }),
      CompileCallbackManager(
          orc::createLocalCompileCallbackManager(TM->getTargetTriple(), 0)),
      CODLayer(OptimizeLayer,
               [this](Function &F) { return std::set<Function*>({&F}); },
               *CompileCallbackManager,
               orc::createLocalIndirectStubsManagerBuilder(
                 TM->getTargetTriple())) {
  llvm::sys::DynamicLibrary::LoadLibraryPermanently(nullptr);
}
```
> 接下来，我们必须更新构造函数以初始化新成员。要创建适当的编译回调管理器，我们使用createLocalCompileCallbackManager函数，如果它接收到编译未知函数的请求，则会调用TargetMachine和JITTargetAddress。在我们简单的JIT中，这种情况不太可能出现，所以我们会欺骗并在这里传递'0'。在生产质量JIT中，您可以提供抛出异常的函数的地址，以便展开JIT代码的堆栈。

> 现在我们可以构造我们的CompileOnDemandLayer。遵循先前层的模式，我们首先将引用传递给堆栈中的下一层 - OptimizeLayer。接下来我们需要提供一个'分区函数'：当一个尚未编译的函数被调用时，CompileOnDemandLayer将调用这个函数来询问我们想要编译什么。至少我们需要编译被调用的函数（由分区函数的参数给出），但我们也可以请求CompileOnDemandLayer编译从被调用的函数无条件调用（或很可能被调用）的其他函数。 。对于KaleidoscopeJIT，我们将保持简单，只需要编译被调用的函数。接下来，我们传递对CompileCallbackManager的引用。最后，我们需要提供一个“间接存根管理器构建器”：一个构造IndirectStubManagers的实用程序函数，它又用于为每个模块中的函数构建存根。对于每次调用addModule，CompileOnDemandLayer都会调用间接存根管理器构建器一次，并使用生成的间接存根管理器为集合中所有模块中的所有函数创建存根。如果/当从JIT中删除模块集时，将删除间接存根管理器，从而释放分配给存根的任何内存。我们使用createLocalIndirectStubsManagerBuilder实用程序提供此功能。对于每次调用addModule，CompileOnDemandLayer都会调用间接存根管理器构建器一次，并使用生成的间接存根管理器为集合中所有模块中的所有函数创建存根。如果/当从JIT中删除模块集时，将删除间接存根管理器，从而释放分配给存根的任何内存。我们使用createLocalIndirectStubsManagerBuilder实用程序提供此功能。对于每次调用addModule，CompileOnDemandLayer都会调用间接存根管理器构建器一次，并使用生成的间接存根管理器为集合中所有模块中的所有函数创建存根。如果/当从JIT中删除模块集时，将删除间接存根管理器，从而释放分配给存根的任何内存。我们使用createLocalIndirectStubsManagerBuilder实用程序提供此功能。
```cpp
// ...
        if (auto Sym = CODLayer.findSymbol(Name, false))
// ...
return cantFail(CODLayer.addModule(std::move(Ms),
                                   std::move(Resolver)));
// ...

// ...
return CODLayer.findSymbol(MangledNameStream.str(), true);
// ...

// ...
CODLayer.removeModule(H);
// ...
```
> 最后，我们需要在addModule，findSymbol和removeModule方法中替换对OptimizeLayer的引用。有了它，我们就开始运转了。

## 3.3。完整的代码清单
> 下面是我们运行示例的完整代码清单，其中添加了CompileOnDemand层以启用惰性函数一次编译。要构建此示例，请使用：
```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Run
./toy
```
> 这是代码：[toy.cpp](./toy.cpp)

[下一篇：极端懒惰 - 直接从AST编译回调到JIT](../Chapter4/README.md)