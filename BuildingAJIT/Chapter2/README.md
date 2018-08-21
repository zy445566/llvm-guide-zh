# 2.构建JIT：添加优化 - ORC层介绍
* 第2章简介
* 使用IRTransformLayer优化模块
* 完整的代码清单

# 2.1。第2章介绍
* 欢迎阅读“在LLVM中构建基于ORC的JIT”教程的第2章。在 本系列的第1章中，我们研究了一个基本的JIT类KaleidoscopeJIT，它可以将LLVM IR模块作为输入并在内存中生成可执行代码。KaleidoscopeJIT能够通过组合两个现成的ORC层来完成相对较少的代码：IRCompileLayer和ObjectLinkingLayer，以完成大部分繁重工作。

* 在这一层中，我们将通过使用新层IRTransformLayer来更多地了解ORC层概念，以便为KaleidoscopeJIT添加IR优化支持。

## 2.2。使用IRTransformLayer优化模块
* 在“使用LLVM实现语言”教程系列的第4章中，使用llvm FunctionPassManager作为优化LLVM IR的手段而引入。感兴趣的读者可以阅读该章节以获取详细信息，但简而言之：为了优化模块，我们创建了一个llvm :: FunctionPassManager实例，使用一组优化对其进行配置，然后在模块上运行PassManager以将其变为（希望）更多优化但语义上等效的形式。在原始教程系列中，FunctionPassManager是在KaleidoscopeJIT之外创建的，模块在添加到模块之前已经过优化。在本章中，我们将优化作为JIT的一个阶段。现在，这将为我们提供更多了解ORC层的动力，但从长远来看，优化JIT的一部分将产生一个重要的好处：当我们开始懒洋洋地编译代码时（即推迟编译每个函数直到它第一次编译）跑），

* 要为我们的JIT添加优化支持，我们将从第1章开始使用KaleidoscopeJIT并在顶部构建 ORC IRTransformLayer。我们将在下面详细介绍IRTransformLayer的工作方式，但界面很简单：该层的构造函数引用下面的层（如同所有层一样）以及它将应用于每个模块的IR优化函数通过addModule添加：
```cpp
class KaleidoscopeJIT {
private:
  std::unique_ptr<TargetMachine> TM;
  const DataLayout DL;
  RTDyldObjectLinkingLayer<> ObjectLayer;
  IRCompileLayer<decltype(ObjectLayer)> CompileLayer;

  using OptimizeFunction =
      std::function<std::shared_ptr<Module>(std::shared_ptr<Module>)>;

  IRTransformLayer<decltype(CompileLayer), OptimizeFunction> OptimizeLayer;

public:
  using ModuleHandle = decltype(OptimizeLayer)::ModuleHandleT;

  KaleidoscopeJIT()
      : TM(EngineBuilder().selectTarget()), DL(TM->createDataLayout()),
        ObjectLayer([]() { return std::make_shared<SectionMemoryManager>(); }),
        CompileLayer(ObjectLayer, SimpleCompiler(*TM)),
        OptimizeLayer(CompileLayer,
                      [this](std::unique_ptr<Module> M) {
                        return optimizeModule(std::move(M));
                      }) {
    llvm::sys::DynamicLibrary::LoadLibraryPermanently(nullptr);
  }
```
> 我们的扩展KaleidoscopeJIT类的开始与第1章中的相同，但在CompileLayer之后，我们为优化函数引入了一个typedef。在这种情况下，我们使用std :: function（一个方便的“函数式”包装器）从一个unique_ptr <Module>输入到std :: unique_ptr <Module>输出。通过我们的优化函数typedef，我们可以声明我们的OptimizeLayer，它位于我们的CompileLayer之上。

> 为了初始化我们的OptimizeLayer，我们将它传递给下面的CompileLayer（层的标准实践），并使用lambda初始化OptimizeFunction，该lambda调用我们将在下面定义的“optimizeModule”函数。
```cpp
// ...
auto Resolver = createLambdaResolver(
    [&](const std::string &Name) {
      if (auto Sym = OptimizeLayer.findSymbol(Name, false))
        return Sym;
      return JITSymbol(nullptr);
    },
// ...
// ...
return cantFail(OptimizeLayer.addModule(std::move(M),
                                        std::move(Resolver)));
// ...
// ...
return OptimizeLayer.findSymbol(MangledNameStream.str(), true);
// ...
// ...
cantFail(OptimizeLayer.removeModule(H));
// ...
```
> 接下来，我们需要在关键方法中使用对OptimizeLayer的引用替换对'CompileLayer'的引用：addModule，findSymbol和removeModule。在addModule中，我们需要小心替换两个引用：我们的解析器中的findSymbol调用，以及对addModule的调用。
```cpp
std::shared_ptr<Module> optimizeModule(std::shared_ptr<Module> M) {
  // Create a function pass manager.
  auto FPM = llvm::make_unique<legacy::FunctionPassManager>(M.get());

  // Add some optimizations.
  FPM->add(createInstructionCombiningPass());
  FPM->add(createReassociatePass());
  FPM->add(createGVNPass());
  FPM->add(createCFGSimplificationPass());
  FPM->doInitialization();

  // Run the optimizations over all functions in the module being added to
  // the JIT.
  for (auto &F : *M)
    FPM->run(F);

  return M;
}
```
> 在我们的JIT底部，我们添加了一个私有方法来进行实际优化： optimizeModule。此函数设置一个FunctionPassManager，向其添加一些传递，在模块中的每个函数上运行它，然后返回变异模块。具体的优化与“使用LLVM实现语言”教程系列的第4章中使用的优化相同 。读者可以访问该章节，以便对这些以及一般的IR优化进行更深入的讨论。

> 这就是对KaleidoscopeJIT的更改：当通过addModule添加模块时，OptimizeLayer会在将转换后的模块传递给下面的CompileLayer之前调用我们的optimizeModule函数。当然，我们可以在我们的addModule函数中直接调用optimizeModule，而不必担心使用IRTransformLayer，但这样做会给我们另一个机会来看看层是如何构成的。它还为层 概念本身提供了一个简洁的入口点，因为IRTransformLayer被证明是可以设计的层概念的最简单实现之一：
```cpp
template <typename BaseLayerT, typename TransformFtor>
class IRTransformLayer {
public:
  using ModuleHandleT = typename BaseLayerT::ModuleHandleT;

  IRTransformLayer(BaseLayerT &BaseLayer,
                   TransformFtor Transform = TransformFtor())
    : BaseLayer(BaseLayer), Transform(std::move(Transform)) {}

  Expected<ModuleHandleT>
  addModule(std::shared_ptr<Module> M,
            std::shared_ptr<JITSymbolResolver> Resolver) {
    return BaseLayer.addModule(Transform(std::move(M)), std::move(Resolver));
  }

  void removeModule(ModuleHandleT H) { BaseLayer.removeModule(H); }

  JITSymbol findSymbol(const std::string &Name, bool ExportedSymbolsOnly) {
    return BaseLayer.findSymbol(Name, ExportedSymbolsOnly);
  }

  JITSymbol findSymbolIn(ModuleHandleT H, const std::string &Name,
                         bool ExportedSymbolsOnly) {
    return BaseLayer.findSymbolIn(H, Name, ExportedSymbolsOnly);
  }

  void emitAndFinalize(ModuleHandleT H) {
    BaseLayer.emitAndFinalize(H);
  }

  TransformFtor& getTransform() { return Transform; }

  const TransformFtor& getTransform() const { return Transform; }

private:
  BaseLayerT &BaseLayer;
  TransformFtor Transform;
};
```
> 这是IRTransformLayer的整个定义，从中 llvm/include/llvm/ExecutionEngine/Orc/IRTransformLayer.h删除了它的注释。它是一个模板类有两个模板参数：BaesLayerT和 TransformFtor提供基础层的分别的类型和“变换函子”（在我们的情况下，一个std ::功能）的类型。这个类涉及两个非常简单的工作：（1）运行通过变换仿函数添加addModule的每个IR模块，以及（2）符合ORC层接口。该接口由一个typedef和五个方法组成：

|接口 |	描述 |
| ------ | ------ | ------ |
|ModuleHandleT |	提供一个句柄，可用于在调用findSymbolIn，removeModule或emitAndFinalize时标识模块集。|
|加入AddModule	 |获取一组给定的模块并使其“可用于执行”。这意味着这些模块中的符号应该可以通过findSymbol和findSymbolIn进行搜索，并且在调用JITSymbol :: getAddress（）之后，符号的地址应该是可读/可写的（对于数据符号）或可执行的（对于函数符号）。注意：这意味着addModule不必预先编译（或执行任何其他工作）。它可以像IRCompileLayer一样急切地行动，但它也可以简单地记录模块，并且在有人调用JITSymbol :: getAddress（）之前不会采取进一步行动。在IRTransformLayer的情况下，addModule急切地将变换函子应用于集合中的每个模块，然后将生成的变异模块集传递到下面的层。|
|removeModule |	从JIT中删除一组模块。这些模块中定义的代码或数据将不再可用，并且将释放包含JIT定义的内存。|
|findSymbol |	在先前通过addModule添加的所有模块中搜索命名符号（但尚未通过调用removeModule删除）。在IRTransformLayer中，我们只是将查询传递给下面的图层。在我们的REPL中，这是我们搜索函数定义的默认方式。|
|findSymbolIn |	在给定的ModuleHandleT指示的模块集中搜索命名符号。这只是一个优化的搜索，当您确切知道应该找到符号定义时，更适合查找速度。在IRTransformLayer中，我们只是将此查询传递给下面的图层。在我们的REPL中，我们使用此方法来搜索表示顶级表达式的函数，因为我们确切地知道它们将在哪里找到它们：在我们刚刚添加的顶级表达式模块中。|
|emitAndFinalize |	强制使模块集（由ModuleHandleT表示）中的代码和数据可访问所需的所有操作。表现为好像在集合中搜索了一些符号并且调用了JITSymbol :: getSymbolAddress。这很少需要，但在处理通常表现懒惰的层时非常有用，如果用户想要触发早期编译（例如，使用空闲CPU时间来急切地在后台编译代码）。|

> 此接口尝试捕获JIT的自然操作（有一些皱纹，例如emitAndFinalize以提高性能），类似于我们在第1章中确定的基本JIT API操作。符合层概念允许类通过以术语实现其行为来巧妙地编写类。这些相同的操作，在下面的层上进行。例如，一个急切的层（如IRTransformLayer）可以通过前面的转换运行集合中的每个模块并立即将结果传递给下面的层来实现addModule。相比之下，一个惰性层可以通过松散模块来实现addModule，而不需要其他前期工作，而是在客户端调用findSymbol时应用转换（并在下面的层上调用addModule）。JIT的程序行为将是相同的，但是这些选择将具有不同的性能特征：急切地工作意味着JIT需要更长的时间，但是一旦完成就可以顺利进行。延迟工作允许JIT快速启动并运行，但是只要需要一些尚未处理的代码或数据，JIT就会暂停并等待。

> 我们当前的REPL非常渴望：每个函数定义在输入后立即进行优化和编译。如果我们要使变换层变得懒惰（但不改变其他东西），我们可以推迟优化，直到我们第一次引用函数顶级表达（看看你能否弄清楚原因，然后看看下面的答案[1]）。在下一章中，我们将介绍完全惰性编译，其中函数在第一次在运行时调用之前不会被编译。在这一点上，权衡变得更加有趣：我们越懒，我们就越能开始执行第一个函数，但是我们不得不暂停编译新遇到的函数。如果我们只是懒惰地编码，但是急切地优化，我们将有一个缓慢的启动（一切都被优化）但相对较短的暂停，因为每个函数只是通过代码。如果我们都懒得优化和代码生成，我们可以更快地开始执行第一个函数，但是我们将有更长的暂停，因为每个函数必须在首次执行时进行优化和代码生成。如果我们考虑像内联这样的跨社会优化，事情会变得更加有趣 必须热切地进行。这些是复杂的权衡，并没有一个适合他们的解决方案，但通过提供可组合层，我们将决策留给实施JIT的人，并使他们可以轻松地尝试不同的配置。

[下一页：添加按功能的惰性编译](../Chapter3/README.md)

## 2.3 完整的代码清单
> 以下是我们运行示例的完整代码清单，其中添加了IRTransformLayer以启用优化。要构建此示例，请使用：

```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Run
./toy
```
> 这是代码：[toy.cpp](./toy.cpp)

* [1]	当我们将顶层表达式添加到JIT时，对我们之前定义的函数的任何调用都将作为外部符号出现在RTDyldObjectLinkingLayer中。RTDyldObjectLinkingLayer将调用我们在addModule中定义的SymbolResolver，后者又调用OptimizeLayer上的findSymbol，此时甚至一个惰性变换层也必须完成它的工作。