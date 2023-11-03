# 香山处理器
## 主要功能

香山处理器是一个开源的工业级别处理器，是乱序六发射结构设计，支持RISC-V指令集，前端流水线负责取指，后端流水线负责处理。

## 主要功能模块

香山处理器前端流水线包括分支预测单元、取指单元、指令缓冲等单元，顺序取指。后端包括译码、重命名、重定序缓冲、保留站、整型/浮点寄存器堆、整型/浮点运算单元。结构如下图：

![xs-arch-nanhu](./img/xs-arch-nanhu.svg)

## 选择分析的功能模块

我选择分析的模块是前端的分支预测器，分支预测单元采用一种多级混合预测的架构，其主要组成部分包括 下一行预测器（Next Line Predictor，以下简称 NLP）和 精确预测器（Accurate Predictor，以下简称 APD）。其中，NLP 是一个 uBTB (micro BTB)，APD 由 FTB1、TAGE-SC、ITTAGE、RAS 组成。NLP 提供无空泡的预测，在预测请求的下一拍就可以得到预测结果。APD 各组成部分的延迟在 2~3 拍之间。其中，FTB、TAGE、RAS 的延迟为 2 拍；SC、ITTAGE 的延迟为 3 拍。一次预测会经历三个流水级，每一个流水级都会产生新的预测内容。这些分布于不同流水级的预测器组成了一个覆盖预测器 (overriding predictor)，结构如下图：

![xs-arch-nanhu](./img/bpu.svg)

香山使用新型敏捷硬件开发语言Chisel，Chisel是一门以Scala为宿主语言开发的硬件构建语言，chisel转换成verilog的过程为：编写的Chisel代码首先经过Chisel编译器生成AST中间数据，然后经过Firrtl编译器，再将AST数据转换成Firrtl代码，最后经过Firrtl编译器将Firrtl文件转换成Verilog代码。