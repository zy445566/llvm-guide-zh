# 4.构建JIT：极端懒惰 - 从AST使用JIT编译回调¶
> 第4章介绍
> 完整的代码清单

## 4.1 第4章介绍
> 欢迎阅读“在LLVM中构建基于ORC的JIT”教程的第4章。本章介绍了Compile Callbacks和Indirect Stubs API，并展示了它们如何用于替换第3章中的CompileOnDemand层 ，使用JIT直接来自Kaleidoscope AST的自定义lazy-JITing方案。

> 要完成：

* （1）描述IR的JITing的缺点（必须首先编译为IR，这会降低懒惰的好处）。

* （2）详细描述CompileCallbackManagers和IndirectStubManagers。

* （3）运行addFunctionAST的实现。

## 4.2 完整的代码清单
> 以下是JIT懒洋洋地从Kaleidoscope ASTS运行的示例的完整代码清单。要构建此示例，请使用：
```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
# Run
./toy
```
> 这是代码：[toy.cpp](./toy.cpp)
[下一页：远程JITing - 过程隔离和远距离懒惰](../Chapter5/README.md)