# 1.构建JIT：从KaleidoscopeJIT开始
* 第1章简介
* JIT API基础知识
* KaleidoscopeJIT
* 完整的代码清单
## 1.1 第1章介绍
> 欢迎阅读“在LLVM中构建基于ORC的JIT”教程的第1章。本教程使用LLVM的请求编译（ORC）API运行JIT编译器的实现。它首先介绍了使用LLVM教程实现语言时使用的KaleidoscopeJIT类的简化版本， 然后介绍了优化，延迟编译和远程执行等新功能。

> 本教程的目标是向您介绍LLVM的ORC JIT API，展示这些API如何与LLVM的其他部分交互，并教您如何重新组合它们以构建适合您的用例的自定义JIT。

> 本教程的结构是：

> 第1章：研究简单的KaleidoscopeJIT类。这将介绍ORC JIT API的一些基本概念，包括ORC 层的概念。
> 第2章：通过添加一个优化IR和生成代码的新层来扩展基本的KaleidoscopeJIT。
> 第3章：通过添加Compile-On-Demand层来进一步扩展JIT以延迟编译IR。
> 第4章：通过直接使用ORC Compile Callbacks API的自定义层替换Compile-On-Demand层来改善JIT的懒惰，以便在调用函数之前推迟IR生成。
> 第5章：使用JIT Remote API通过JITing代码将进程隔离添加到具有降低权限的远程进程中。
为了为我们的JIT提供输入，我们将使用“在LLVM中实现语言教程” 第7章中的Kaleidoscope REPL， 只需稍作修改：我们将从该章的代码中删除FunctionPassManager，并将其替换为我们的优化支持。第2章中的JIT类。

> 最后，关于API代的一句话：ORC是第三代LLVM JIT API。它之前是MCJIT，之前是（现已删除的）遗留JIT。这些教程不假设使用这些早期的API，但熟悉它们的读者会看到许多熟悉的元素。在适当的情况下，我们将使用早期的API显式连接，以帮助从过渡到ORC的人员。

## 1.2 JIT API基础知识
> JIT编译器的目的是在需要时“即时”编译代码，而不是像传统编译器那样提前将整个程序编译到磁盘。为了支持这一目标，我们最初的，简单的JIT API将是：

> 处理addModule（Module＆M） - 使给定的IR模块可用于执行。
> JITSymbol findSymbol（const std :: string＆Name） - 搜索已添加到JIT的符号（函数或变量）的指针。
> void removeModule（Handle H） - 从JIT中删除一个模块，释放已用于编译代码的所有内存。
> 此API的基本用例，从模块执行'main'函数，如下所示：
```cpp
std::unique_ptr<Module> M = buildModule();
JIT J;
Handle H = J.addModule(*M);
int (*Main)(int, char*[]) = (int(*)(int, char*[]))J.getSymbolAddress("main");
int Result = Main();
J.removeModule(H);
```
> 我们在这些教程中构建的API都将是这个简单主题的变体。在API的背后，我们将优化JIT的实现，以增加对优化和延迟编译的支持。最终，我们将扩展API本身，以允许将更高级别的程序表示（例如AST）添加到JIT。

## 1.3 KaleidoscopeJIT 
> 在上一节中，我们描述了我们的API，现在我们检查它的一个简单实现：KaleidoscopeJIT类[1]，它用于 实现具有LLVM的语言教程。我们将使用第7章中的REPL代码该教程为我们的JIT提供输入：每次用户输入表达式时，REPL都会将包含该表达式代码的新IR模块添加到JIT中。如果表达式是顶级表达式，如'1 + 1'或'sin（x）'，则REPL也将使用我们的JIT类的findSymbol方法查找并执行表达式的代码，然后使用removeModule方法再次删除代码（因为没有办法重新调用匿名表达式）。在本教程的后续章节中，我们将修改REPL以启用与JIT类的新交互，但是现在我们将此设置视为理所当然，并将注意力集中在JIT本身的实现上。

> 我们的KaleidoscopeJIT类在KaleidoscopeJIT.h头文件中定义。在通常包括警卫和#includes [2]之后，我们得到了我们班级的定义：
```cpp
#ifndef LLVM_EXECUTIONENGINE_ORC_KALEIDOSCOPEJIT_H
#define LLVM_EXECUTIONENGINE_ORC_KALEIDOSCOPEJIT_H

#include "llvm/ADT/STLExtras.h"
#include "llvm/ExecutionEngine/ExecutionEngine.h"
#include "llvm/ExecutionEngine/JITSymbol.h"
#include "llvm/ExecutionEngine/RTDyldMemoryManager.h"
#include "llvm/ExecutionEngine/SectionMemoryManager.h"
#include "llvm/ExecutionEngine/Orc/CompileUtils.h"
#include "llvm/ExecutionEngine/Orc/IRCompileLayer.h"
#include "llvm/ExecutionEngine/Orc/LambdaResolver.h"
#include "llvm/ExecutionEngine/Orc/RTDyldObjectLinkingLayer.h"
#include "llvm/IR/DataLayout.h"
#include "llvm/IR/Mangler.h"
#include "llvm/Support/DynamicLibrary.h"
#include "llvm/Support/raw_ostream.h"
#include "llvm/Target/TargetMachine.h"
#include <algorithm>
#include <memory>
#include <string>
#include <vector>

namespace llvm {
namespace orc {

class KaleidoscopeJIT {
private:
  std::unique_ptr<TargetMachine> TM;
  const DataLayout DL;
  RTDyldObjectLinkingLayer ObjectLayer;
  IRCompileLayer<decltype(ObjectLayer), SimpleCompiler> CompileLayer;

public:
  using ModuleHandle = decltype(CompileLayer)::ModuleHandleT;
```

> 我们的类有四个成员：一个TargetMachine，TM，它将用于构建我们的LLVM编译器实例; DataLayout，DL，将用于符号修改（稍后将详细介绍）和两个ORC 层：RTDyldObjectLinkingLayer和CompileLayer。我们将在下一章中更多地讨论层，但是现在您可以将它们视为类似于LLVM Passes：它们将易用的组合接口背后的有用JIT实用程序包装起来。第一层ObjectLayer是我们JIT的基础：它接收由编译器生成的内存中的目标文件，并动态链接它们以使它们可执行。这个JIT-on-of-a-linker设计是在MCJIT中引入的，但链接器隐藏在MCJIT类中。在ORC中，我们公开链接器，以便客户端可以在需要时直接访问和配置它。在本教程中，我们的ObjectLayer将仅用于支持堆栈中的下一层：CompileLayer，它负责获取LLVM IR，编译它，

> 这就是成员变量，之后我们有一个typedef：ModuleHandle。这是将从我们的JIT的addModule方法返回的句柄类型，并且可以传递给removeModule方法以删除模块。IRCompileLayer类已经提供了一个方便的句柄类型（IRCompileLayer :: ModuleHandleT），所以我们只是将我们的ModuleHandle别名。
```cpp
KaleidoscopeJIT()
    : TM(EngineBuilder().selectTarget()), DL(TM->createDataLayout()),
      ObjectLayer([]() { return std::make_shared<SectionMemoryManager>(); }),
      CompileLayer(ObjectLayer, SimpleCompiler(*TM)) {
  llvm::sys::DynamicLibrary::LoadLibraryPermanently(nullptr);
}

TargetMachine &getTargetMachine() { return *TM; }
```
> 接下来我们有我们的类构造函数。我们首先使用EngineBuilder :: selectTarget辅助方法初始化TM，该方法为当前进程构造TargetMachine。然后我们使用新创建的TargetMachine初始化DL，我们的DataLayout。之后我们需要初始化ObjectLayer。ObjectLayer需要一个函数对象，它将为添加的每个模块构建一个JIT内存管理器（JIT内存管理器管理内存分配，内存权限和JIT代码的异常处理程序注册）。为此，我们使用lambda返回一个SectionMemoryManager，这是一个现成的实用程序，提供本章所需的所有基本内存管理功能。接下来我们初始化我们的CompileLayer。CompileLayer需要两件事：（1）对象层的引用，（2）用于执行从IR到目标文件的实际编译的编译器实例。我们现在使用现成的SimpleCompiler实例。最后，在构造函数的主体中，我们使用nullptr参数调用DynamicLibrary :: LoadLibraryPermanently方法。通常使用要加载的动态库的路径调用LoadLibraryPermanently方法，但是当传递空指针时，它将“加载”主机进程本身，使其导出的符号可用于执行。
```cpp
ModuleHandle addModule(std::unique_ptr<Module> M) {
  // Build our symbol resolver:
  // Lambda 1: Look back into the JIT itself to find symbols that are part of
  //           the same "logical dylib".
  // Lambda 2: Search for external symbols in the host process.
  auto Resolver = createLambdaResolver(
      [&](const std::string &Name) {
        if (auto Sym = CompileLayer.findSymbol(Name, false))
          return Sym;
        return JITSymbol(nullptr);
      },
      [](const std::string &Name) {
        if (auto SymAddr =
              RTDyldMemoryManager::getSymbolAddressInProcess(Name))
          return JITSymbol(SymAddr, JITSymbolFlags::Exported);
        return JITSymbol(nullptr);
      });

  // Add the set to the JIT with the resolver we created above and a newly
  // created SectionMemoryManager.
  return cantFail(CompileLayer.addModule(std::move(M),
                                         std::move(Resolver)));
}
```
> 现在我们来看看第一个JIT API方法：addModule。此方法负责将IR添加到JIT并使其可用于执行。在我们的JIT的初始实现中，我们将通过将它们直接添加到CompileLayer来使我们的模块“可用于执行”，CompileLayer将立即编译它们。在后面的章节中，我们将教我们的JIT推迟单个函数的编译，直到它们被实际调用。

> 要将我们的模块添加到CompileLayer，我们需要提供模块和符号解析器。符号解析器负责为JIT提供每个外部符号的地址在我们正在添加的模块中。外部符号是模块本身未定义的任何符号，包括对JIT外部函数的调用以及对已添加到JIT的其他模块中定义的函数的调用。（似乎添加到JIT的模块默认情况下应该彼此了解，但由于我们仍然必须提供符号解析器来引用JIT之外的代码，因此更容易重用这一机制对于所有符号分辨率。）这具有额外的好处，即用户可以完全控制符号解析过程。我们应该首先在JIT中搜索定义，然后回到外部定义？或者，如果我们还没有可用的实现，我们是否应该选择可用的外部定义，而只选择JIT代码？通过使用单一符号解析方案，我们可以自由选择对任何给定用例最有意义的内容。

> createLambdaResolver 函数使构建符号解析器变得特别容易。这个函数需要两个lambdas [3]并返回一个JITSymbolResolver实例。第一个lambda用作解析器的findSymbolInLogicalDylib方法的实现，该方法搜索符号定义，这些符号定义应被视为与此模块相同的“逻辑”动态库的一部分。如果您熟悉静态链接：这意味着findSymbolInLogicalDylib应该公开具有公共链接和隐藏可见性的符号。如果所有这些听起来都是外来的，你可以忽略细节，只记得这是链接器用来尝试查找符号定义的第一种方法。如果findSymbolInLogicalDylib方法返回null结果，则链接器将调用名为findSymbol的第二个符号解析器方法，该方法搜索应该被认为是模块及其逻辑dylib外部（但可见）的符号。在本教程中，我们将采用以下简单方案：添加到JIT的所有模块的行为就像它们被链接到一个不断增长的逻辑dylib中一样。为了实现这个，我们的第一个lambda（定义findSymbolInLogicalDylib的那个）将通过调用CompileLayer的findSymbol方法来搜索JIT代码。如果我们在JIT本身找不到符号，我们将回到我们的第二个lambda，它实现了findSymbol。这将使用RTDyldMemoryManager :: getSymbolAddressInProcess方法在程序本身中搜索符号。如果我们无法通过这些路径找到符号定义，JIT将拒绝接受我们的模块，返回“未找到符号”错误。为了实现这个，我们的第一个lambda（定义findSymbolInLogicalDylib的那个）将通过调用CompileLayer的findSymbol方法来搜索JIT代码。如果我们在JIT本身找不到符号，我们将回到我们的第二个lambda，它实现了findSymbol。这将使用RTDyldMemoryManager :: getSymbolAddressInProcess方法在程序本身中搜索符号。如果我们无法通过这些路径找到符号定义，JIT将拒绝接受我们的模块，返回“未找到符号”错误。为了实现这个，我们的第一个lambda（定义findSymbolInLogicalDylib的那个）将通过调用CompileLayer的findSymbol方法来搜索JIT代码。如果我们在JIT本身找不到符号，我们将回到我们的第二个lambda，它实现了findSymbol。这将使用RTDyldMemoryManager :: getSymbolAddressInProcess方法在程序本身中搜索符号。如果我们无法通过这些路径找到符号定义，JIT将拒绝接受我们的模块，返回“未找到符号”错误。getSymbolAddressInProcess方法，用于搜索程序本身内的符号。如果我们无法通过这些路径找到符号定义，JIT将拒绝接受我们的模块，返回“未找到符号”错误。getSymbolAddressInProcess方法，用于搜索程序本身内的符号。如果我们无法通过这些路径找到符号定义，JIT将拒绝接受我们的模块，返回“未找到符号”错误。

> 现在我们已经构建了符号解析器，我们已准备好将我们的模块添加到JIT中。我们通过调用CompileLayer的addModule方法来完成此操作。addModule方法返回一个Expected<CompileLayer::ModuleHandle>，因为在更高级的JIT配置中它可能会失败。在我们的基本配置中，我们知道它将始终成功，因此我们使用cantFail实用程序断言没有发生错误，并提取句柄值。由于我们已经将我们的ModuleHandle类型设置为与CompileLayer的句柄类型相同，我们可以直接返回未包装的句柄。
```cpp
JITSymbol findSymbol(const std::string Name) {
  std::string MangledName;
  raw_string_ostream MangledNameStream(MangledName);
  Mangler::getNameWithPrefix(MangledNameStream, Name, DL);
  return CompileLayer.findSymbol(MangledNameStream.str(), true);
}

JITTargetAddress getSymbolAddress(const std::string Name) {
  return cantFail(findSymbol(Name).getAddress());
}

void removeModule(ModuleHandle H) {
  cantFail(CompileLayer.removeModule(H));
}
```
> 既然我们可以向JIT添加代码，我们需要一种方法来查找我们添加到它的符号。要做到这一点，我们呼吁我们的CompileLayer的findSymbol方法，但有一个转折：我们必须裂伤我们正在寻找第一个符号的名称。ORC JIT组件在内部使用受损的符号，与静态编译器和链接器的使用方式相同，而不是使用纯IR符号名称。这允许JIT代码与应用程序或共享库中的预编译代码轻松互操作。修改的类型将取决于DataLayout，而DataLayout又取决于目标平台。为了让我们能够保持便携性并根据未损坏的名称进行搜索，我们自己重新制作了这个。

> 接下来我们有一个便利函数getSymbolAddress，它返回给定符号的地址。与CompileLayer的addModule函数一样，JITSymbol的getAddress函数被允许失败[4]，但是我们知道它不会在我们的简单示例中，所以我们将它包装在对cantFail的调用中。

> 我们现在来到JIT API中的最后一个方法：removeModule。此方法负责销毁随给定模块添加的MemoryManager和SymbolResolver，从而释放它们在进程中使用的任何资源。在我们的Kaleidoscope演示中，我们依靠此方法来移除表示最新顶级表达式的模块，从而防止在输入下一个顶级表达式时将其视为重复定义。通常情况下，释放任何您不需要进一步调用的模块，只需释放专用的资源即可。但是，您并不需要这样做：当您的JIT类被破坏时，如果在此之前尚未释放它们，则将清除所有资源。喜欢 CompileLayer::addModule和JITSymbol::getAddress，removeModule一般可能会失败，但在我们的示例中永远不会失败，所以我们将它包装在对cantFail的调用中。

> 这将我们带到构建JIT的第1章的末尾。您现在拥有一个基本但功能齐全的JIT堆栈，您可以使用它来获取LLVM IR并使其在JIT进程的上下文中可执行。在下一章中，我们将介绍如何扩展此JIT以生成更高质量的代码，并在此过程中深入了解ORC层概念。

[下一步：扩展KaleidoscopeJIT](../Chapter2/README.md)

## 1.4。完整的代码清单
> 以下是我们正在运行的示例的完整代码清单。要构建此示例，请使用：
```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Run
./toy
```
> 这是代码：[toy.cpp](./toy.cpp)

* [1]	实际上，我们使用KaleidoscopeJIT的缩减版本，这是一个简化的假设：符号无法重新定义。这将使得无法在REPL中重新定义符号，但会使我们的符号查找逻辑更简单。重新引入对符号重新定义的支持留给读者练习。（原始教程中使用的KaleidoscopeJIT.h将是一个有用的参考）。
* [2]	
文件	包含的原因
STLExtras.h	在使用STL时很有用的LLVM实用程序。
ExecutionEngine.h	访问EngineBuilder :: selectTarget方法。
RTDyldMemoryManager.h	访问RTDyldMemoryManager :: getSymbolAddressInProcess方法。
CompileUtils.h	提供SimpleCompiler类。
IRCompileLayer.h	提供IRCompileLayer类。
LambdaResolver.h	访问createLambdaResolver函数，该函数提供了符号解析器的简单构造。
RTDyldObjectLinkingLayer.h	提供RTDyldObjectLinkingLayer类。
Mangler.h	为平台特定的名称修改提供Mangler类。
DynamicLibrary.h	提供DynamicLibrary类，使主机进程中的符号可搜索。
raw_ostream.h	快速输出流类。我们使用raw_string_ostream子类进行符号修改
TargetMachine.h	LLVM目标机器描述类。
* [3]	实际上它们不必是lambdas，任何带有调用操作符的对象都可以，包括普通旧函数或std ::函数。
* [4]	JITSymbol::getAddress 将强制JIT编译符号的定义（如果尚未编译），并且由于编译过程可能失败，getAddress必须能够返回此失败。