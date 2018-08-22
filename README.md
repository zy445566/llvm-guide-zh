# LLVM入门教程
* 该版本未经官方允许
* 请勿做任何商用
* 借助了谷歌翻译，可能存在不正确的语序
* 每个教程目录下都有对应源码 ，gitbook的目录已经写好，大家可以down下来转gitbook来方便自己阅读

## 万花筒：用LLVM实现语言（备注：万花筒(Kaleidoscope)是LLVM实现的语言名称）

* [万花筒：教程简介和Lexer](./Chapter01/README.md)
* [万花筒：实现解析器和AST](./Chapter02/README.md)
* [万花筒：代码生成到LLVM IR](./Chapter03/README.md)
* [万花筒：添加JIT和优化器支持](./Chapter04/README.md)
* [万花筒：扩展语言：控制流程](./Chapter05/README.md)
* [万花筒：扩展语言：用户定义的运算符](./Chapter06/README.md)
* [万花筒：扩展语言：可变变量](./Chapter07/README.md)
* [万花筒：编译为目标代码](./Chapter08/README.md)
* [万花筒：添加调试信息](./Chapter09/README.md)
* [万花筒：结论和其他有用的LLVM花絮](./Chapter10/README.md)

## 在LLVM中构建JIT
* [构建JIT：从KaleidoscopeJIT开始](./BuildingAJIT/Chapter1/README.md)
* [构建JIT：添加优化 - ORC层的介绍](./BuildingAJIT/Chapter2/README.md)
* [构建JIT：按函数惰性编译](./BuildingAJIT/Chapter3/README.md)
* [构建JIT：极端懒惰 - 使用从AST编译JIT的编译回调](./BuildingAJIT/Chapter4/README.md)
* [构建JIT：远程JITing - 远程处理隔离和懒惰](./BuildingAJIT/Chapter5/README.md)
