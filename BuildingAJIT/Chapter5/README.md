# 5.构建JIT：远程JITing - 远程处理隔离和懒惰
* 第5章简介
* 完整的代码清单

## 5.1 第5章介绍
> 欢迎阅读“在LLVM中构建基于ORC的JIT”教程的第5章。本章介绍ORC RemoteJIT客户端/服务器API，并说明如何使用它们构建JIT堆栈，该堆栈将通过具有不同进程的通信通道执行其代码。这可以是同一台机器上的单独进程，不同机器上的进程，甚至是不同平台/体系结构上的进程。代码构建在第4章的惰性AST编译JIT堆栈之上。

> 要做 - 这将是一个漫长的过程：

* （1）介绍渠道，RPC，RemoteJIT客户端和服务器API

* （2）更详细地描述客户端代码。讨论KaleidoscopeJIT类和REPL本身的修改。

* （3）描述服务器代码。

* （4）描述如何运行演示。

## 5.2 完整的代码清单
> 以下是JIT懒洋洋地从Kaleidoscope ASTS运行的示例的完整代码清单。要构建此示例，请使用：
```sh
# Compile
clang++ -g toy.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy
clang++ -g Server/server.cpp `llvm-config --cxxflags --ldflags --system-libs --libs core orcjit native` -O3 -o toy-server
# Run
./toy-server &
./toy
```
修改后的KaleidoscopeJIT的代码：[KaleidoscopeJIT.h](./KaleidoscopeJIT.h)
以及JIT服务器的代码：[server.cpp](./Server/server.cpp)