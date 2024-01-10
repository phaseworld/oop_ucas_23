# 香山处理器
## 1. 主要功能

香山处理器是一个开源的工业级别处理器，是乱序六发射结构设计，支持RISC-V指令集，前端流水线负责取指，后端流水线负责处理。

阅读版本为：南湖架构。

## 2. 主要功能模块

香山处理器前端流水线包括分支预测单元、取指单元、指令缓冲等单元，顺序取指。后端包括译码、重命名、重定序缓冲、保留站、整型/浮点寄存器堆、整型/浮点运算单元。结构如下图：

![xs-arch-nanhu](./img/xs-arch-nanhu.svg)

香山官方文档目录结构如下：
    
```bash
.
├── scripts              # 生成 Verilog 及仿真使用的一些脚本
├── src                  # 结构设计与验证代码
│   └── main               # 结构设计代码
│       └── scala
│           ├── device         # 仿真用的一些外设
│           ├── system         # SoC 的描述
│           ├── top            # 顶层文件
│           ├── utils          # 一些基础硬件工具库
│           ├── xiangshan      # 香山 CPU 部分的设计代码
│           └── xstransforms   # 一些 FIRRTL Transforms
├── fudian               # 香山浮点子模块
├── huancun              # 香山 L2/L3 缓存子模块
├── difftest             # 香山协同仿真框架
├── ready-to-run         # 预先编译好的 nemu 动态链接库, 和一些负载
└── rocket-chip          # 用来获取 Diplomacy 框架（等待上游拆分）
```

```./src/main/scala/xiangshan```包含了CPU设计代码，笔者阅读的主要是```src/main/scala/utils```和```src/main/scala/xiangshan```。

```./src/main/scala/xiangshan```目录如下：

```bash
xiangshan
├── backend
├── cache
├── frontend
└── mem
```

```frontend```目录包含了前端流水线的设计代码，其中就包括笔者要分析的分支预测单元。

## 3. 选择分析的功能模块

笔者选择分析的模块是前端的分支预测器，分支预测单元采用一种多级混合预测的架构，其主要组成部分包括 下一行预测器（Next Line Predictor，以下简称 NLP）和 精确预测器（Accurate Predictor，以下简称 APD）。其中，NLP 是一个 uBTB (micro BTB)，APD 由 FTB1、TAGE-SC、ITTAGE、RAS 组成。NLP 提供无空泡的预测，在预测请求的下一拍就可以得到预测结果。APD 各组成部分的延迟在 2~3 拍之间。其中，FTB、TAGE、RAS 的延迟为 2 拍；SC、ITTAGE 的延迟为 3 拍。一次预测会经历三个流水级，每一个流水级都会产生新的预测内容。这些分布于不同流水级的预测器组成了一个覆盖预测器 (overriding predictor)，结构如下图：

![xs-arch-nanhu](./img/bpu.svg)

香山使用新型敏捷硬件开发语言Chisel，Chisel是一门以Scala为宿主语言开发的硬件构建语言，chisel转换成verilog的过程为：编写的Chisel代码首先经过Chisel编译器生成AST中间数据，然后经过Firrtl编译器，再将AST数据转换成Firrtl代码，最后经过Firrtl编译器将Firrtl文件转换成Verilog代码。

## 4. 分析香山源码面向对象思想

香山使用新型敏捷硬件开发语言Chisel，Chisel是一门以Scala为宿主语言开发的硬件构造语言，是面向对象语言，香山的开发利用了Chisel的面向对象特性，实现了敏捷开发。

### 4.1 Generators: Parameters

Verilog中的模块都是静态的，即模块的端口数量和类型都是固定的，而Chisel中的模块依托于Scala的类，因此可以使用Scala的参数化特性，使得模块的端口数量和类型可以在运行时确定，这样的模块被称为生成器，利用生成器可以大大减少重复建造相同类型不同规模的轮子。

在香山的工具库中大量使用了参数：

```Scala
class RawDataModuleTemplate[T <: Data](
  gen: T,           // 数据类型
  numEntries: Int,  // 模块的端口数量
  numRead: Int,     // 读端口数量
  numWrite: Int,    // 写端口数量
  isSync: Boolean,  // 是否同步
  optWrite: Seq[Int] = Seq()    // 可选写端口
) extends Module {
  val io = IO(new Bundle {
    val rvec  = Vec(numRead,  Input(UInt(numEntries.W)))
    val rdata = Vec(numRead,  Output(gen))
    val wen   = Vec(numWrite, Input(Bool()))
    val wvec  = Vec(numWrite, Input(UInt(numEntries.W)))
    val wdata = Vec(numWrite, Input(gen))
  })

  val data = Reg(Vec(numEntries, gen))

  val wen = io.wen.zipWithIndex.map{ case (en, i) => if (optWrite.contains(i)) RegNext(en) else en }
  val wvec = io.wvec.zipWithIndex.map{ case (v, i) => if (optWrite.contains(i)) RegEnable(v, io.wen(i)) else v }
  val wdata = io.wdata.zipWithIndex.map{ case (d, i) => if (optWrite.contains(i)) RegEnable(d, io.wen(i)) else d }

  // read ports
    /* 此处省略读端口代码 */
  // write ports
    /* 此处省略写端口代码 */
}
```

### 4.2 Chisel的函数式编程

高阶函数指的是以函数作为参数或返回值的函数，用Scala提供的高阶函数(higher-order function),我们能将原本Verilog代码大量重复且晦涩难懂的代码，简洁地表示出来。

```Scala
object ParallelOperation {
  def apply[T](xs: Seq[T], func: (T, T) => T): T = {    // func为传入的函数参数
    require(xs.nonEmpty)
    xs match {
      case Seq(a) => a
      case Seq(a, b) => func(a, b)
      case _ =>
        apply(Seq(apply(xs take xs.size/2, func), apply(xs drop xs.size/2, func)), func)
    }
  }
}

object ParallelOR {
  def apply[T <: Data](xs: Seq[T]): T = {
    ParallelOperation(xs, (a: T, b: T) => (a.asUInt | b.asUInt).asTypeOf(xs.head))  // 传入的函数参数为(a: T, b: T) => (a.asUInt | b.asUInt).asTypeOf(xs.head)，是一个函数字面量。
  }
}
```

Scala中函数为头等公民，我们可以写一个未命名的函数字面量，然后可以把它当成一个值传递到其它函数或是赋值给其它变量。

### 4.3 封装与信息隐藏

CPU的设计是一个复杂的系统工程，需要对各个模块进行封装，封装的目的是为了隐藏模块的内部实现细节，使得模块的使用者不需要关心模块的内部实现细节，只需要关心模块的接口，例如：CPU并不关心BPU的内部实现细节，只关心预测结果，因此BPU的内部实现细节对CPU隐藏，只暴露预测结果给CPU。

```Scala
class FrontendImp (outer: Frontend, parentName:String = "Unknown") extends LazyModuleImp(outer)
  with HasXSParameter
  with HasPerfEvents
{
  val io = IO(new Bundle() {
    val reset_vector = Input(UInt(PAddrBits.W))
    val fencei = Input(Bool())
    val ptw = new TlbPtwIO(6)
    val backend = new FrontendToCtrlIO
    val sfence = Input(new SfenceBundle)
    val tlbCsr = Input(new TlbCsrBundle)
    val csrCtrl = Input(new CustomCSRCtrlIO)
    val csrUpdate = new DistributedCSRUpdateReq
    val error  = new L1CacheErrorInfo
    val frontendInfo = new Bundle {
      val ibufFull  = Output(Bool())
      val bpuInfo = new Bundle {
        val bpRight = Output(UInt(XLEN.W))
        val bpWrong = Output(UInt(XLEN.W))
      }
    }
  })

  // bpu ctrl
  bpu.io.reset_vector := io.reset_vector
  bpu.io.ctrl := csrCtrl.bp_ctrl

  //IFU-Ftq
  ifu.io.ftqInter.fromFtq <> ftq.io.toIfu
  ftq.io.toIfu.req.ready :=  ifu.io.ftqInter.fromFtq.req.ready && icache.io.fetch.req.ready

  ftq.io.fromIfu          <> ifu.io.ftqInter.toFtq
  bpu.io.ftq_to_bpu       <> ftq.io.toBpu
  ftq.io.fromBpu          <> bpu.io.bpu_to_ftq

  ftq.io.mmioCommitRead   <> ifu.io.mmioCommitRead
}
```

CPU负责调用BPU的接口，BPU的内部实现细节对CPU隐藏，只暴露预测结果给CPU。

### 4.4 Chisel的面向对象编程

Chisel可以使用其宿主语言Scala的面向对象编程特征，类、抽象类(abstract class)和特质(trait)。类和抽象类我们都比较熟悉，特质则抽象类有所不同：1.一个类可以继承多个特质。2.特质不能有构造函数参数，这意味着特质不能实例化。

```Scala
class Predictor(parentName:String = "Unknown")(implicit p: Parameters) extends XSModule with HasBPUConst with HasPerfEvents with HasCircularQueuePtrHelper {  
  // 继承了HasBPUConst、HasPerfEvents、HasCircularQueuePtrHelper三个特质，同时香山的硬件模块继承XSModule这个抽象类。
/* 此处省略代码 */
}
```

Scala为单例类提供了一个称为object的语言特性。不能实例化一个object，可以直接引用它。这使它们类似于Java静态类。

Scala中伴随对象(copanion object)也很常见，当一个类和一个对象共享相同的名称并在同一个文件中定义时，该对象称为伴随对象。在class/object名称之前使用new时，它将实例化类。如果不使用new，它将引用对象。伴随对象通常用于定义工厂方法。

```Scala

class ImmExtractor(numSrc: Int, dataBits: Int)(implicit p: Parameters) extends XSModule {
  val io = IO(new Bundle {
    val uop = Input(new MicroOp)
    val data_in = Vec(numSrc, Input(UInt(dataBits.W)))
    val data_out = Vec(numSrc, Output(UInt(dataBits.W)))
  })
  io.data_out := io.data_in
}

object ImmExtractor {
  def apply(params: RSParams, uop: MicroOp, data_in: Vec[UInt], pc: Option[UInt], target: Option[UInt])
           (implicit p: Parameters): Vec[UInt] = {
    val immExt = if (params.isJump) {
      val ext = Module(new JumpImmExtractor)
      ext.jump_pc := pc.get
      ext.jalr_target := target.get
      ext
    }
    else if (params.isAlu) { Module(new AluImmExtractor) }
    else if (params.isMul) { Module(new MduImmExtractor) }
    else if (params.isLoad) { Module(new LoadImmExtractor) }
    else { Module(new ImmExtractor(params.numSrc, params.dataBits)) }
    immExt.io.uop := uop
    immExt.io.data_in := data_in
    immExt.io.data_out
  }
}

class JumpImmExtractor(implicit p: Parameters) extends ImmExtractor(2, 64) {
  val jump_pc = IO(Input(UInt(VAddrBits.W)))
  val jalr_target = IO(Input(UInt(VAddrBits.W)))

  when (SrcType.isPc(io.uop.ctrl.srcType(0))) {
    io.data_out(0) := SignExt(jump_pc, XLEN)
  }
  // when src1 is reg (like sfence's asid) do not let data_out(1) be the jalr_target
  when (SrcType.isPcOrImm(io.uop.ctrl.srcType(1))) {
    io.data_out(1) := jalr_target
  }
}
```

```Scala
class FTBMeta(implicit p: Parameters) extends XSBundle with FTBParams {
  val writeWay = UInt(log2Ceil(numWays).W)
  val hit = Bool()
  val fauFtbHit = if (EnableFauFTB) Some(Bool()) else None
  val pred_cycle = if (!env.FPGAPlatform) Some(UInt(64.W)) else None
}

object FTBMeta {
  def apply(writeWay: UInt, hit: Bool, fauhit: Bool, pred_cycle: UInt)(implicit p: Parameters): FTBMeta = {
    val e = Wire(new FTBMeta)
    e.writeWay := writeWay
    e.hit := hit
    e.fauFtbHit.map(_ := fauhit)
    e.pred_cycle.map(_ := pred_cycle)
    e
  }
}
```

### 4.5 单一职责原则

单一职责原则是指一个类只负责一个功能领域中的相应职责，或者可以定义为：就一个类而言，应该只有一个引起它变化的原因。比如每一个分支预测器只负责一种分支预测方法，不负责其他功能，这样可以使得模块的功能更加单一，更加容易维护，做到高内聚低耦合。

```Scala
  branchPredictor: Function3[BranchPredictionResp, Parameters, String, Tuple2[Seq[BasePredictor], BranchPredictionResp]] =
    ((resp_in: BranchPredictionResp, p: Parameters, parentName: String) => {
      val ftb = Module(new FTB(parentName = parentName + "ftb_")(p))
      val ubtb =Module(new FauFTB()(p))
      // val bim = Module(new BIM()(p))
      val tage = Module(new Tage_SC(parentName = parentName + "tage_")(p))
      val ras = Module(new RAS(parentName = parentName + "ras_")(p))
      val ittage = Module(new ITTage(parentName = parentName + "ittage_")(p))
      /* 此处省略代码 */
    })
```

### 4.6 多态

各种分支预测器都继承自抽象类```BasePredictor```，而继承自同一模板，自然会有一些通过重写父类方法来实现多态的需求，比如```BasePredictor```中的```meta_size```和```is_fast_pred```两个变量，分别代表预测器的元数据大小和预测器是否是快速预测器，而这两个变量的值在各个分支预测器中是不同的，因此需要重写父类的这两个变量。

```Scala
class FauFTB(implicit p: Parameters) extends BasePredictor with FauFTBParams {
  
  class FauFTBMeta(implicit p: Parameters) extends XSBundle with FauFTBParams {
  /* 此处省略代码 */
  }
  val resp_meta = Wire(new FauFTBMeta)
  override val meta_size = resp_meta.getWidth
  override val is_fast_pred = true // fast predictor
  class FauFTBBank(implicit p: Parameters) extends XSModule with FauFTBParams {
  /* 此处省略代码 */
  }
  /* 此处省略代码 */
  override val perfEvents = Seq(
  ("fauftb_commit_hit       ", us(0).valid &&  u_meta_dup(0).hit),
  ("fauftb_commit_miss      ", us(0).valid && !u_meta_dup(0).hit),
)
}
```

多态提高了代码的可维护性，提高了代码的扩展性。

## 5. 香山分支预测器设计分析

### 5.1 顶层模块

香山分支预测器的顶层模块在```src\main\scala\xiangshan\frontend\BPU.scala```，名称是```Predictor```

```Scala
class Predictor(parentName:String = "Unknown")(implicit p: Parameters) extends XSModule with HasBPUConst with HasPerfEvents with HasCircularQueuePtrHelper {  // 继承了HasBPUConst、HasPerfEvents、HasCircularQueuePtrHelper三个特质，同时香山的硬件模块都继承了XSModule这个抽象类。
  val io = IO(new PredictorIO)

  val ctrl = DelayN(io.ctrl, 1)
  val predictors = Module(if (useBPD) new Composer(parentName = parentName + "predictors_")(p) else new FakePredictor)
  /* 此处省略代码 */
}
```

在```HasBPUConst```这个特质中，有一个参数```useBPD```，它是一个布尔值，用来控制是否使用分支预测器，如果不使用分支预测器，那么就使用```FakePredictor```，如果使用分支预测器就例化各个分支预测器。

而每个分支预测器都继承自一个抽象类，名称是```BasePredictor```。

### 5.2 BasePredictor

这个抽象类的代码如下：

```Scala
abstract class BasePredictor(parentName:String = "Unknown")(implicit p: Parameters) extends XSModule
  with HasBPUConst with BPUUtils with HasPerfEvents {
  val meta_size = 0
  val spec_meta_size = 0
  val is_fast_pred = false
  val io = IO(new BasePredictorIO())
  /* 此处省略代码 */
  }
```

这个抽象类定义了预测器的接口标准和一些常数的值，比如```is_fast_pred```代表这预测器能不能在一拍之内完成预测，默认值是不能。在硬件开发中，这能大大减少工程人员不断的翻看工程开发手册来定义开发模块的接口，这是面向对象带来的好处。

下面就是一些预测器的介绍。

### 5.3 FakePredictor

```Scala
class FakePredictor(implicit p: Parameters) extends BasePredictor {
  io.in.ready                 := true.B
  io.out.last_stage_meta      := 0.U
  io.out := io.in.bits.resp_in(0)
}
```

这个类是在```useBPD```为false时，也就是不使用分支预测器的时候使用的分支预测器，这个分支预测器预测结果是所有分支都不跳转，这跟我们做的没有分支预测器的流水线是等价的。

### 5.4 FauFTB

FauFTB香山使用的最简单的一个分支预测器，在预测器的工厂branchPredictor方法中，它的例化名称是ubtb，它是快速预测器，也就是说只需要一个时钟周期就能返回分支预测的结果，FauFTB的代码如下:

```Scala
class FauFTB(implicit p: Parameters) extends BasePredictor with FauFTBParams {
  
  class FauFTBMeta(implicit p: Parameters) extends XSBundle with FauFTBParams {
  /* 此处省略代码 */
  }
  val resp_meta = Wire(new FauFTBMeta)
  override val meta_size = resp_meta.getWidth
  override val is_fast_pred = true // fast predictor
  class FauFTBBank(implicit p: Parameters) extends XSModule with FauFTBParams {
  /* 此处省略代码 */
  }
  /* 此处省略代码 */
}
```

### 5.5 FTB

FTB是香山使用的分支预测器，FTB并不是一个快速预测器，它需要两个时钟周期才能返回分支预测的结果，FTB的代码如下：

```Scala
class FTB(parentName:String = "Unknown")(implicit p: Parameters) extends BasePredictor(parentName)(p) with FTBParams with BPUUtils
  with HasCircularQueuePtrHelper with HasPerfEvents {
  override val meta_size = WireInit(0.U.asTypeOf(new FTBMeta)).getWidth

  val ftbAddr = new TableAddr(log2Up(numSets), 1)

  class FTBBank(val numSets: Int, val nWays: Int) extends XSModule with BPUUtils {
  /* 此处省略代码 */
  } // FTBBank

  val ftbBank = Module(new FTBBank(numSets, numWays))

  override val perfEvents = Seq(
    ("ftb_commit_hits            ", u.valid  &&  u_meta.hit),
    ("ftb_commit_misses          ", u.valid  && !u_meta.hit),
  )
  generatePerfEvent()
}
```

可以看到FTB继承自BasePredictor，而且拥有多个特质，在FTB中定义了一个FTBBank类，同时定义了一个ftbBank对象，这个对象是FTBBank类的实例化对象，而且在统计性能事件的时候，采用了多态的方式，重写了perfEvents变量。

### 5.5 例化预测器的流程

香山的前端模块中的```Frontend```类中例化了```FrontendImp```类：

```Scala
class Frontend(parentName:String = "Unknown")(implicit p: Parameters) extends LazyModule with HasXSParameter{
  /* 此处省略代码 */
  lazy val module = new FrontendImp(this,parentName)
  /* 此处省略代码 */
}
```
在```FrontendImp```类中例化了```Predictor```类

```Scala
class FrontendImp (outer: Frontend, parentName:String = "Unknown") extends LazyModuleImp(outer)
  with HasXSParameter
  with HasPerfEvents
{
  /* 此处省略代码 */
  val bpu     = Module(new Predictor(parentName = parentName + "bpu_")(p))
  /* 此处省略代码 */
}
```

在```Predictor```类中例化了```Composer```类

```Scala
class Predictor(parentName:String = "Unknown")(implicit p: Parameters) extends XSModule with HasBPUConst with HasPerfEvents with HasCircularQueuePtrHelper {
  /* 此处省略代码 */
  val predictors = Module(if (useBPD) new Composer(parentName = parentName + "predictors_")(p) else new FakePredictor)
  /* 此处省略代码 */
}
```

在```Composer```类中用```getBPDComponents```获得要例化的预测器的参数，并使用XSCoreParameters中的 ```branchPredictor```方法，该方法是分支预测器的工厂方法，完成各个预测器的实例化。

```Scala
class Composer(parentName:String = "Unknown")(implicit p: Parameters) extends BasePredictor with HasBPUConst with HasPerfEvents {
  val (components, resp) = getBPDComponents(io.in.bits.resp_in(0), p, parentName = parentName)
  /* 此处省略代码 */
}

  def getBPDComponents(resp_in: BranchPredictionResp, p: Parameters, parentName:String = "Unknown") = {
    coreParams.branchPredictor(resp_in, p, parentName)
  }

  branchPredictor: Function3[BranchPredictionResp, Parameters, String, Tuple2[Seq[BasePredictor], BranchPredictionResp]] =
    ((resp_in: BranchPredictionResp, p: Parameters, parentName: String) => {
      val ftb = Module(new FTB(parentName = parentName + "ftb_")(p))
      val ubtb =Module(new FauFTB()(p))
      // val bim = Module(new BIM()(p))
      val tage = Module(new Tage_SC(parentName = parentName + "tage_")(p))
      val ras = Module(new RAS(parentName = parentName + "ras_")(p))
      val ittage = Module(new ITTage(parentName = parentName + "ittage_")(p))
      /* 此处省略代码 */
    })
```

### 5.6 分支预测器类关系图


![xiangshan类图](./img/xiangshan类图.png)

各个分支预测器继承自抽象类```BasePredictor```，```Predictor```类继承自```XSModule```类，同时Predictor类拥有一个```Composer```类的实例化对象，```Composer```类继承自```BasePredictor```类，```Composer```类中有一个```getBPDComponents```方法，该方法调用了```XSCoreParameters```类中的```branchPredictor```方法，该方法是分支预测器的工厂方法，生成完成各个预测器的实例化。

## 6. 设计模式

### 6.1 适配器模式

适配器模式是将一个类的接口转换成客户希望的另外一个接口，使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

在硬件设计中，我们常常要使用一些协议，如果这些协议的接口与我们设计的模块的接口不兼容，我们就需要使用适配器模式，将这些协议的接口转换成我们需要的接口：转接桥。

其实分支预测器的顶层模块```Predictor```就是一个适配器模式的体现，它的IO接口如下：

```Scala
class Predictor(parentName:String = "Unknown")(implicit p: Parameters) extends XSModule with HasBPUConst with HasPerfEvents with HasCircularQueuePtrHelper {
  val io = IO(new PredictorIO)  // 统一的预测器IO
  val predictors = Module(if (useBPD) new Composer(parentName = parentName + "predictors_")(p) else new FakePredictor)

  /* 此处省略代码 */
  io.bpu_to_ftq.resp.bits.s3.hasRedirect.zip(s3_redirect_dup).map {case (hr, r) => hr := r}
  io.bpu_to_ftq.resp.bits.s3.ftq_idx := s3_ftq_idx

  predictors.io.update := RegNext(dup(io.ftq_to_bpu.update))
  predictors.io.update.map(_.bits.ghist := RegNext(getHist(io.ftq_to_bpu.update.bits.spec_info.histPtr)))
  /* 此处省略代码 */
}
```

```Predictor```中的io模块和它例化的具体预测器predictors.io互相交互，向顶层模块提供统一的预测器IO。

### 6.2 工厂模式

最典型的就是在 4.3 中提到的伴随对象，伴随对象通常用于定义工厂方法。

```Scala

class ImmExtractor(numSrc: Int, dataBits: Int)(implicit p: Parameters) extends XSModule {
  val io = IO(new Bundle {
    val uop = Input(new MicroOp)
    val data_in = Vec(numSrc, Input(UInt(dataBits.W)))
    val data_out = Vec(numSrc, Output(UInt(dataBits.W)))
  })
  io.data_out := io.data_in
}

object ImmExtractor {
  def apply(params: RSParams, uop: MicroOp, data_in: Vec[UInt], pc: Option[UInt], target: Option[UInt])
           (implicit p: Parameters): Vec[UInt] = {
    val immExt = if (params.isJump) {
      val ext = Module(new JumpImmExtractor)
      ext.jump_pc := pc.get
      ext.jalr_target := target.get
      ext
    }
    else if (params.isAlu) { Module(new AluImmExtractor) }
    else if (params.isMul) { Module(new MduImmExtractor) }
    else if (params.isLoad) { Module(new LoadImmExtractor) }
    else { Module(new ImmExtractor(params.numSrc, params.dataBits)) }
    /* 此处省略代码 */
  }
}

class JumpImmExtractor(implicit p: Parameters) extends ImmExtractor(2, 64) {
  val jump_pc = IO(Input(UInt(VAddrBits.W)))
  val jalr_target = IO(Input(UInt(VAddrBits.W)))
  /* 此处省略代码 */
}
```

IO其实也一个伴随对象，它的apply方法接收一个Bundle类型的参数, 返回一个IO类实例化的对象，apply是一个静态方法，apply方法可以省略，直接用对象名加括号调用。



