# chisel学习笔记
## 1. intro_to_scala
### (1). Variables and Constants

Variables and Constants前缀分别是var和val，尽量用val。

Scala并不一定需要分号，因为Scala会根据换行符来判断语句是否结束，但是如果一行中有多个语句，就需要用分号隔开。

### (2). Conditions

```scala
if (condition) {
  // do something
} else {
  // do something else
}
```

除了有else而且单行语句情况下，if语句可以不用大括号。

一个if语句可以返回值，返回值取决于选择的分支，可用来初始化函数和类的值。

```scala
val likelyCharactersSet = if (alphabet.length == 26)
    "english"
else 
    "not english"

println(likelyCharactersSet)
```

### (3). Methods(Functions)

方法使用def关键字定义，返回类型应该明确指定，无参函数可以省略括号，如果无参函数有副作用，应该加上括号。

```scala
// Simple scaling function with an input argument, e.g., times2(3) returns 6
// Curly braces can be omitted for short one-line functions.
def times2(x: Int): Int = 2 * x

// More complicated function
def distance(x: Int, y: Int, returnPositive: Boolean): Int = {
    val xy = x * y
    if (returnPositive) xy.abs else -xy.abs
}
```

函数可以被重载，即同名函数可以有不同的参数列表，但并不推荐这么做。

```scala
// Overloaded function
def times2(x: Int): Int = 2 * x
def times2(x: String): Int = 2 * x.toInt

times2(5)
times2("7")
```

可以在函数内定义函数，这种函数称为局部函数，局部函数只能在定义它的函数内部使用，函数内部可以递归的调用自己。

```scala
/** Prints a triangle made of "X"s
  * This is another style of comment
  */
def asciiTriangle(rows: Int) {
    
    // This is cute: multiplying "X" makes a string with many copies of "X"
    def printRow(columns: Int): Unit = println("X" * columns)
    
    if(rows > 0) {
        printRow(rows)
        asciiTriangle(rows - 1) // Here is the recursive call
    }
}

// printRow(1) // This would not work, since we're calling printRow outside its scope
asciiTriangle(6)
```
### (4)Lists
Scala实现了List，List很像数组，支持append和extract。

```scala
val x = 7
val y = 14
val list1 = List(1, 2, 3)
val list2 = x :: y :: y :: Nil       // An alternate notation for assembling a list

val list3 = list1 ++ list2           // Appends the second list to the first list
val m = list2.length
val s = list2.size

val headOfList = list1.head          // Gets the first element of the list
val restOfList = list1.tail          // Get a new list with first element removed

val third = list1(2)                 // Gets the third element of a list (0-indexed)
```

### (4)for State

to和until都是Range的方法，to包含最后一个元素，until不包含最后一个元素。

```scala
for (i <- 0 to 7) { print(i + " ") }
println()// 0 1 2 3 4 5 6 7
for (i <- 0 until 7) { print(i + " ") }
println()// 0 1 2 3 4 5 6
```

by是Range的方法，可以指定步长。

```scala
for(i <- 0 to 10 by 2) { print(i + " ") }
println()// 0 2 4 6 8 10
```

遍历list

```scala
val randomList = List(scala.util.Random.nextInt(), scala.util.Random.nextInt(), scala.util.Random.nextInt(), scala.util.Random.nextInt())
var listSum = 0
for (value <- randomList) {
  listSum += value
}
println("sum is " + listSum)
```

### (5)Packages and Imports

```scala
import chisel3._
import chisel3.iotesters.{ChiselFlatSpec, Driver, PeekPokeTester}
```

上面第一行导入了chisel3包的所有内容，第二行导入了chisel3.iotesters包中的ChiselFlatSpec、Driver、PeekPokeTester三个类。

### (6)Classes

```scala
// WrapCounter counts up to a max value based on a bit size
class WrapCounter(counterBits: Int) {

  val max: Long = (1 << counterBits) - 1
  var counter = 0L
    
  def inc(): Long = {
    counter = counter + 1
    if (counter > max) {
        counter = 0
    }
    counter
  }
  println(s"counter created with max value $max")
}
```

方法中最后一行的counter是返回值，最后的println是构造函数的一部分，会在对象创建时执行。s表明字符串是可以插值的，$表示插入变量，如果要插入表达式，需要用${}，如果返回值不是字符串，它会被转换成字符串。

### (7)Creating an Instance of a Class

```scala
val x = new WrapCounter(2)
x.inc() // Increments the counter

// Member variables of the instance x are visible to the outside, unless they are declared private
if(x.counter == x.max) {              
    println("counter is about to wrap")
}

x inc() // Scala allows the dots to be omitted; this can be useful for making embedded DSL's look more natural
```

Scala允许省略点，这对于嵌入式DSL来说很有用。

### (8)Code Blocks

需要注意代码块可以作为参数传递给List的map方法，map方法要求代码块只有一个参数。

```scala
val intList = List(1, 2, 3)
val stringList = intList.map { i =>
  i.toString
}// List("1", "2", "3")
```
### (9)Named Parameters and Parameter Defaults

Unit是Scala中的void，Unit是一个类型，只有一个值，即()。

函数参数可以有默认值，如果函数有多个参数，可以指定参数名来指定参数。

```scala
def myMethod(count: Int, wrap: Boolean, wrapValue: Int = 24): Unit = { ... }
myMethod(wrapValue = 23, wrap = false, count = 10)
myMethod(wrap = false, count = 10)
```

## 2.1 chisel3
### (1)First Chisel Module

```scala
// Chisel Code: Declare a new module definition
class Passthrough extends Module {// Module is a class provided by Chisel
  val io = IO(new Bundle {// 必须叫io，是IO的实例
    val in = Input(UInt(4.W))// UInt是无符号整数，4.W是4位宽
    val out = Output(UInt(4.W))// Output是输出，Input是输入
  })
  io.out := io.in// :=表明右边的信号驱动左边的信号
}
```

```scala
// Chisel Code, but pass in a parameter to set widths of ports
class PassthroughGenerator(width: Int) extends Module { 
  val io = IO(new Bundle {
    val in = Input(UInt(width.W))
    val out = Output(UInt(width.W))
  })
  io.out := io.in
}

// Let's now generate modules with different widths
println(getVerilog(new PassthroughGenerator(10)))
println(getVerilog(new PassthroughGenerator(20)))
```

可以通过Scala的特性进行参数化设计，这种类型被称为generator。

### (2)Testing Chisel Modules

```scala
// Scala Code: `test` runs the unit test. 
// test takes a user Module and has a code block that applies pokes and expects to the 
// circuit under test (c)
test(new Passthrough()) { c =>
    c.io.in.poke(0.U)     // Set our input to value 0
    c.io.out.expect(0.U)  // Assert that the output correctly has 0
    c.io.in.poke(1.U)     // Set our input to value 1
    c.io.out.expect(1.U)  // Assert that the output correctly has 1
    c.io.in.poke(2.U)     // Set our input to value 2
    c.io.out.expect(2.U)  // Assert that the output correctly has 2
}
println("SUCCESS!!") // Scala Code: if we get here, our tests passed!
```

poke是设置输入，expect是断言输出，也可以用peek查看输出。

### (3)Generating Verilog

```scala
// Viewing the Verilog for debugging
println(getVerilog(new Passthrough))
```

```scala
class PrintingModule extends Module {
    val io = IO(new Bundle {
        val in = Input(UInt(4.W))
        val out = Output(UInt(4.W))
    })
    io.out := io.in

    printf("Print during simulation: Input is %d\n", io.in)
    // chisel printf has its own string interpolator too
    printf(p"Print during simulation: IO is $io\n")

    println(s"Print during generation: Input is ${io.in}")
}

test(new PrintingModule ) { c =>
    c.io.in.poke(3.U)
    c.clock.step(5) // circuit will print
    
    println(s"Print during testing: Input is ${c.io.in.peek()}")
}
```

printf是在仿真时打印，println是在生成时打印，printf有自己的字符串插值器。step是时钟步进，参数是步进的周期数，这时候会进行printf。

## 2.2 combinational_logic
### (1)Common Operators

```scala
class MyModule extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(4.W))
    val out = Output(UInt(4.W))
  })

  val two  = 1 + 1
  println(two)
  val utwo = 1.U + 1.U
  println(utwo)
  
  io.out := io.in
}
println(getVerilog(new MyModule))
```

two是scala的Int，utwo是chisel的UInt，1.U+1不兼容，Scala是强类型语言，类型必须显式声明。

所有Chisel变量被声明为Scala的val，不要使用var作为硬件设计。

减、乘：

```Scala
class MyOperators extends Module {
  val io = IO(new Bundle {
    val in      = Input(UInt(4.W))
    val out_add = Output(UInt(4.W))
    val out_sub = Output(UInt(4.W))
    val out_mul = Output(UInt(4.W))
  })

  io.out_add := 1.U + 4.U
  io.out_sub := 2.U - 1.U
  io.out_mul := 4.U * 2.U
}
println(getVerilog(new MyOperators))
```

Mux、Concatenation

```Scala
class MyOperatorsTwo extends Module {
  val io = IO(new Bundle {
    val in      = Input(UInt(4.W))
    val out_mux = Output(UInt(4.W))
    val out_cat = Output(UInt(4.W))
  })

  val s = true.B
  io.out_mux := Mux(s, 3.U, 0.U) // should return 3.U, since s is true
  io.out_cat := Cat(2.U, 1.U)    // concatenates 2 (b10) with 1 (b1) to give 5 (101)
}

println(getVerilog(new MyOperatorsTwo))

test(new MyOperatorsTwo) { c =>
  c.io.out_mux.expect(3.U)
  c.io.out_cat.expect(5.U)
}
println("SUCCESS!!")
```

但是Verilog并没有生成这些组合逻辑，只是生成结果常量，这是因为FIRRTL transformations简化了电路，消除了明显的逻辑。

MAC乘加：
```Scala
class MAC extends Module {
  val io = IO(new Bundle {
    val in_a = Input(UInt(4.W))
    val in_b = Input(UInt(4.W))
    val in_c = Input(UInt(4.W))
    val out  = Output(UInt(8.W))
  })

  io.out := (io.in_a * io.in_b) + io.in_c
}

test(new MAC) { c =>
  val cycles = 100
  import scala.util.Random
  for (i <- 0 until cycles) {
    val in_a = Random.nextInt(16)
    val in_b = Random.nextInt(16)
    val in_c = Random.nextInt(16)
    c.io.in_a.poke(in_a.U)
    c.io.in_b.poke(in_b.U)
    c.io.in_c.poke(in_c.U)
    c.io.out.expect((in_a * in_b + in_c).U)
  }
}
println("SUCCESS!!")
```

### (2) Parameterized Adder

```Scala
class ParameterizedAdder(saturate: Boolean) extends Module {
  val io = IO(new Bundle {
    val in_a = Input(UInt(4.W))
    val in_b = Input(UInt(4.W))
    val out  = Output(UInt(4.W))
  })

    val sum = io.in_a +& io.in_b
    if(saturate)
        io.out := Mux(sum > 15.U, 15.U, sum)
    else
        io.out := sum 
}

for (saturate <- Seq(true, false)) {
  test(new ParameterizedAdder(saturate)) { c =>
    // 100 random tests
    val cycles = 100
    import scala.util.Random
    import scala.math.min
    for (i <- 0 until cycles) {
      val in_a = Random.nextInt(16)
      val in_b = Random.nextInt(16)
      c.io.in_a.poke(in_a.U)
      c.io.in_b.poke(in_b.U)
      if (saturate) {
        c.io.out.expect(min(in_a + in_b, 15).U)
      } else {
        c.io.out.expect(((in_a + in_b) % 16).U)
      }
    }
    
    // ensure we test saturation vs. truncation
    c.io.in_a.poke(15.U)
    c.io.in_b.poke(15.U)
    if (saturate) {
      c.io.out.expect(15.U)
    } else {
      c.io.out.expect(14.U)
    }
  }
}
println("SUCCESS!!")
```

"+"会产生的结果位宽是两个操作数中较大的那个，"+&"会产生的结果位宽是两个操作数中较大的那个加1。直接把多位宽的信号赋值给少位宽的信号会发生截断。参数化并不是生成一个截断加法器或者一个饱和加法器，而是生成一个加法器，然后根据参数选择是否饱和，这在编译时确定。

## 2.3 control_flow

### (1)Last Connect Semantics

```Scala
class LastConnect extends Module {
  val io = IO(new Bundle {
    val in = Input(UInt(4.W))
    val out = Output(UInt(4.W))
  })
  io.out := 1.U
  io.out := 2.U
  io.out := 3.U
  io.out := 4.U
}

//  Test LastConnect
test(new LastConnect) { c => c.io.out.expect(4.U) } // Assert that the output correctly has 4
println("SUCCESS!!") // Scala Code: if we get here, our tests passed!
```

最后一次赋值会覆盖之前的赋值，这种行为称为Last Connect Semantics。

### (2)when, elsewhen, and otherwise

chisel中主要的控制流语句是when，elsewhen，otherwise，伪码如下：

```Scala
when(someBooleanCondition) {
  // things to do when true
}.elsewhen(someOtherBooleanCondition) {
  // things to do on this condition
}.otherwise {
  // things to do if none of th boolean conditions are true
}
```

<div style="display: none">
They must appear in the above order, though either of the latter may be omitted. There can be as many elsewhen clauses as desired. Any section that is true terminates the construct. Actions taken in the bodies of the the three can be complex blocks and may contain nested when and allies. Unlike Scala if, values are not returned by the blocks associated with when.
</div>

这些语句必须按照上面的顺序出现，后两个可以省略，elsewhen可以有多个。任何一个条件为真，就会终止这个结构。when的块可以是复杂的块，可以包含嵌套的when和其他语句。不像Scala的if，when的块不会返回值。

```Scala
// Max3 returns the max of its 3 arguments
class Max3 extends Module {
  val io = IO(new Bundle {
    val in1 = Input(UInt(16.W))
    val in2 = Input(UInt(16.W))
    val in3 = Input(UInt(16.W))
    val out = Output(UInt(16.W))
  })
    
  when(io.in1 >= io.in2 && io.in1 >= io.in3) {
    io.out := io.in1  
  }.elsewhen(io.in2 >= io.in3) {
    io.out := io.in2 
  }.otherwise {
    io.out := io.in3
  }
}
```

### (3)The Wire Construct

Wire定义了一个电路元件，它可以用:=赋值

```Scala
/** Sort4 sorts its 4 inputs to its 4 outputs */
class Sort4 extends Module {
  val io = IO(new Bundle {
    val in0 = Input(UInt(16.W))
    val in1 = Input(UInt(16.W))
    val in2 = Input(UInt(16.W))
    val in3 = Input(UInt(16.W))
    val out0 = Output(UInt(16.W))
    val out1 = Output(UInt(16.W))
    val out2 = Output(UInt(16.W))
    val out3 = Output(UInt(16.W))
  })

  val row10 = Wire(UInt(16.W))
  val row11 = Wire(UInt(16.W))
  val row12 = Wire(UInt(16.W))
  val row13 = Wire(UInt(16.W))

  when(io.in0 < io.in1) {
    row10 := io.in0            // preserve first two elements
    row11 := io.in1
  }.otherwise {
    row10 := io.in1            // swap first two elements
    row11 := io.in0
  }

  when(io.in2 < io.in3) {
    row12 := io.in2            // preserve last two elements
    row13 := io.in3
  }.otherwise {
    row12 := io.in3            // swap last two elements
    row13 := io.in2
  }

  val row21 = Wire(UInt(16.W))
  val row22 = Wire(UInt(16.W))

  when(row11 < row12) {
    row21 := row11            // preserve middle 2 elements
    row22 := row12
  }.otherwise {
    row21 := row12            // swap middle two elements
    row22 := row11
  }

  val row20 = Wire(UInt(16.W))
  val row23 = Wire(UInt(16.W))
  when(row10 < row13) {
    row20 := row10            // preserve middle 2 elements
    row23 := row13
  }.otherwise {
    row20 := row13            // swap middle two elements
    row23 := row10
  }

  when(row20 < row21) {
    io.out0 := row20            // preserve first two elements
    io.out1 := row21
  }.otherwise {
    io.out0 := row21            // swap first two elements
    io.out1 := row20
  }

  when(row22 < row23) {
    io.out2 := row22            // preserve first two elements
    io.out3 := row23
  }.otherwise {
    io.out2 := row23            // swap first two elements
    io.out3 := row22
  }
}
```

一个用List实现的test：

```Scala
// Here's the tester
test(new Sort4) { c =>
  // verify the all possible ordering of 4 numbers are sorted
  List(1, 2, 3, 4).permutations.foreach { case i0 :: i1 :: i2 :: i3 :: Nil =>// permutations是List的方法,返回所有排列,case i0 :: i1 :: i2 :: i3 :: Nil是模式匹配
    println(s"Sorting $i0 $i1 $i2 $i3")
    c.io.in0.poke(i0.U)
    c.io.in1.poke(i1.U)
    c.io.in2.poke(i2.U)
    c.io.in3.poke(i3.U)
    c.io.out0.expect(1.U)
    c.io.out1.expect(2.U)
    c.io.out2.expect(3.U)
    c.io.out3.expect(4.U)
  }
}
println("SUCCESS!!") // Scala Code: if we get here, our tests passed!
```

chesel判等是用的===，Scala判等是用的==，Scala的==是比较值，chesel的===是比较信号。

::表示普通元素与List的连接操作。

## 2.4 sequential_logic

### (1)Registers

Reg是一个寄存器，在时钟上升沿时，它的值会被更新，每一个chisel的Module都有一个共享的隐式时钟，隐式时钟可以被重写以满足多时钟设计的需求。

```Scala
class RegisterModule extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(12.W))// W表示位宽
    val out = Output(UInt(12.W))
  })
  
  val register = Reg(UInt(12.W))
  register := io.in + 1.U
  io.out := register
}

test(new RegisterModule) { c =>
  for (i <- 0 until 100) {
    c.io.in.poke(i.U)
    c.clock.step(1)
    c.io.out.expect((i + 1).U)
  }
}
println("SUCCESS!!")
```

Reg(tpe),tpe是Reg的类型参数，参数必须是类型(UInt等)，但不能是硬件节点(2.U)。测试时，需要时钟步进，step(n)表示步进n个周期。

```Scala
class RegNextModule extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(12.W))
    val out = Output(UInt(12.W))
  })
  
  // register bitwidth is inferred from io.out
  io.out := RegNext(io.in + 1.U)
}

test(new RegNextModule) { c =>
  for (i <- 0 until 100) {
    c.io.in.poke(i.U)
    c.clock.step(1)
    c.io.out.expect((i + 1).U)
  }
}
println("SUCCESS!!")
```

RegNext的位宽是根据io.out推断的，RegNext的参数必须是硬件节点。

### (2)RegInit

RegInit会在reset时初始化寄存器。
  
  ```Scala
  class RegInitModule extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(12.W))
    val out = Output(UInt(12.W))
  })
  
  val register = RegInit(0.U(12.W))
  // val register = RegInit(UInt(12.W), 0.U) // equivalent to above
  register := io.in + 1.U
  io.out := register
}

println(getVerilog(new RegInitModule))
```

chisel隐式的reset信号是高有效、同步的，PeekPokeTester会在每个测试之前自动重置电路，也可以手动重置电路，reset(n)表示重置n个周期。

### (3)Control Flow

```Scala
class FindMax extends Module {
  val io = IO(new Bundle {
    val in  = Input(UInt(10.W))
    val max = Output(UInt(10.W))
  })

  val max = RegInit(0.U(10.W))
  when (io.in > max) {
    max := io.in
  }
  io.max := max
}

test(new FindMax) { c =>
    c.io.max.expect(0.U)
    c.io.in.poke(1.U)
    c.clock.step(1)
    c.io.max.expect(1.U)
    c.io.in.poke(3.U)
    c.clock.step(1)
    c.io.max.expect(3.U)
    c.io.in.poke(2.U)
    c.clock.step(1)
    c.io.max.expect(3.U)
    c.io.in.poke(24.U)
    c.clock.step(1)
    c.io.max.expect(24.U)
}
println("SUCCESS!!")
```

```Scala
class Comb extends Module {
  val io = IO(new Bundle {
    val in  = Input(SInt(12.W))
    val out = Output(SInt(12.W))
  })

  val delay: SInt = Reg(SInt(12.W))
  delay := io.in
  io.out := io.in - delay
}
println(getVerilog(new Comb))
```

会生成一个名称为delay的寄存器，delay的值是io.in，io.out的值是io.in减去delay。

### (4)Appendix: Explicit Clock and Reset

withClock(){},withReset(){},withClockAndReset(){}可以重写隐式时钟和重置信号,reset总是同步的而且是Bool类型

## 2.5 chiseltest

### (1)Basic Tester implementation

|          | iotesters                  | ChiselTest            |
| :------- | :------------------------- | :-------------------- |
| poke     | poke(c.io.in1, 6)          | c.io.in1.poke(6.U)    |
| peek     | peek(c.io.out1)            | c.io.out1.peek()      |
| expect   | expect(c.io.out1, 6)       | c.io.out1.expect(6.U) |
| step     | step(1)                    | c.io.clock.step(1)    |
| initiate | Driver.execute(...) { c => | test(...) { c =>      |

一些例子：

```Scala
val testResult = Driver(() => new Passthrough()) {
  c => new PeekPokeTester(c) {
    poke(c.io.in, 0)     // Set our input to value 0
    expect(c.io.out, 0)  // Assert that the output correctly has 0
    poke(c.io.in, 1)     // Set our input to value 1
    expect(c.io.out, 1)  // Assert that the output correctly has 1
    poke(c.io.in, 2)     // Set our input to value 2
    expect(c.io.out, 2)  // Assert that the output correctly has 2
  }
}
assert(testResult)   // Scala Code: if testResult == false, will throw an error
println("SUCCESS!!") // Scala Code: if we get here, our tests passed!
```

```Scala
test(new PassthroughGenerator(16)) { c =>
    c.io.in.poke(0.U)     // Set our input to value 0
    c.clock.step(1)    // advance the clock
    c.io.out.expect(0.U)  // Assert that the output correctly has 0
    c.io.in.poke(1.U)     // Set our input to value 1
    c.clock.step(1)    // advance the clock
    c.io.out.expect(1.U)  // Assert that the output correctly has 1
    c.io.in.poke(2.U)     // Set our input to value 2
    c.clock.step(1)    // advance the clock
    c.io.out.expect(2.U)  // Assert that the output correctly has 2
}
```

### (2)Modules with Decoupled Interfaces

```Scala
class QueueModule[T <: Data](ioType: T, entries: Int) extends MultiIOModule {
  val in = IO(Flipped(Decoupled(ioType)))
  val out = IO(Decoupled(ioType))
  out <> Queue(in, entries)
}
```

QueueModule传递一个类型参数T，T必须是Data的子类，in是一个Decoupled的输入，out是一个Decoupled的输出，out和Queue的输出连接。

| method           | description                                                        |
| :--------------- | :----------------------------------------------------------------- |
| enqueueNow       | Add (enqueue) one element to a `Decoupled` input interface         |
| expectDequeueNow | Removes (dequeues) one element from a `Decoupled` output interface |
---

Decoupled（解耦）用法：Decoupled接口是一个与握手信号Valid和Ready有关的接口。Producer（生产者）驱动数据(bits)和Valid线，Consumer（消费者）驱动Ready线。当生产者产生的数据准备就绪时，生产者将Valid信号拉高。当消费者准备接收数据时，消费者把Ready信号拉高。如果Valid和Ready信号同时处在高电平时，生产者和消费者发生交火（fire）。

```Scala
test(new QueueModule(UInt(9.W), entries = 200)) { c =>
    // Example testsequence showing the use and behavior of Queue
    c.in.initSource()
    c.in.setSourceClock(c.clock)
    c.out.initSink()
    c.out.setSinkClock(c.clock)
    
    val testVector = Seq.tabulate(200){ i => i.U }

    testVector.zip(testVector).foreach { case (in, out) =>
      c.in.enqueueNow(in)
      c.out.expectDequeueNow(out)
    }
}
```

initSource()和initSink()初始化输入和输出，setSourceClock()和setSinkClock()设置时钟，enqueueNow()和expectDequeueNow()分别是入队和出队,这是必须的来确保ready和valid都被正确的初始化。

| method           | description                                                                                                                            |
| :--------------- | :------------------------------------------------------------------------------------------------------------------------------------- |
| enqueueSeq       | Continues to add (enqueue) elements from the `Seq` to a `Decoupled` input interface, one at a time, until the sequence is exhausted    |
| expectDequeueSeq | Removes (dequeues) elements from a `Decoupled` output interface, one at a time, and compares each one to the next element of the `Seq` |
---

要注意的是，enqueueSeq()必须在expectDequeueSeq()之前调用，而且集合大小必须小于等于队列的大小。

### (3)Fork and Join in ChiselTest

| method | description                                                                                                                                                   |
| :----- | :------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| fork   | launches a concurrent code block, additional forks can be run concurrently to this one via the .fork appended to end of the code block of the preceeding fork |
| join   | re-unites multiple related forks back into the calling thread                                                                                                 |
---

```Scala
test(new QueueModule(UInt(9.W), entries = 200)) { c =>
    // Example testsequence showing the use and behavior of Queue
    c.in.initSource()
    c.in.setSourceClock(c.clock)
    c.out.initSink()
    c.out.setSinkClock(c.clock)
    
    val testVector = Seq.tabulate(300){ i => i.U }

    fork {
        c.in.enqueueSeq(testVector)
    }.fork {
        c.out.expectDequeueSeq(testVector)
    }.join()
}
```

## 3.1 Generators: Parameters

### (1)Motivation

参数丰富度和生成电路复杂度成正比，参数应该提供有用的默认值，应该易于设置，防止非法或无意义的参数值。对于更复杂的系统，如果它们能够以一种不会无意中影响其他模块使用的方式进行本地重写，那么它们将非常有用。

### (2)Parameter Passing

类似于Scala传参。

```Scala
class ParameterizedWidthAdder(in0Width: Int, in1Width: Int, sumWidth: Int) extends Module {
  require(in0Width >= 0)
  require(in1Width >= 0)
  require(sumWidth >= 0)
  val io = IO(new Bundle {
    val in0 = Input(UInt(in0Width.W))
    val in1 = Input(UInt(in1Width.W))
    val sum = Output(UInt(sumWidth.W))
  })
  // a +& b includes the carry, a + b does not
  io.sum := io.in0 +& io.in1
}

println(getVerilog(new ParameterizedWidthAdder(1, 4, 6)))
```

require()是生成前检查参数的函数，assert()是仿真时检查参数的函数。

```Scala
/** Sort4 sorts its 4 inputs to its 4 outputs */
class Sort4(ascending: Boolean) extends Module {
  val io = IO(new Bundle {
    val in0 = Input(UInt(16.W))
    val in1 = Input(UInt(16.W))
    val in2 = Input(UInt(16.W))
    val in3 = Input(UInt(16.W))
    val out0 = Output(UInt(16.W))
    val out1 = Output(UInt(16.W))
    val out2 = Output(UInt(16.W))
    val out3 = Output(UInt(16.W))
  })
    
  // this comparison funtion decides < or > based on the module's parameterization
  def comp(l: UInt, r: UInt): Bool = {
      if (ascending) {
        l < r
      } else {
        l > r
    }
  }

  val row10 = Wire(UInt(16.W))
  val row11 = Wire(UInt(16.W))
  val row12 = Wire(UInt(16.W))
  val row13 = Wire(UInt(16.W))

  when(comp(io.in0, io.in1)) {
    row10 := io.in0            // preserve first two elements
    row11 := io.in1
  }.otherwise {
    row10 := io.in1            // swap first two elements
    row11 := io.in0
  }

  when(comp(io.in2, io.in3)) {
    row12 := io.in2            // preserve last two elements
    row13 := io.in3
  }.otherwise {
    row12 := io.in3            // swap last two elements
    row13 := io.in2
  }

  val row21 = Wire(UInt(16.W))
  val row22 = Wire(UInt(16.W))

  when(comp(row11, row12)) {
    row21 := row11            // preserve middle 2 elements
    row22 := row12
  }.otherwise {
    row21 := row12            // swap middle two elements
    row22 := row11
  }

  val row20 = Wire(UInt(16.W))
  val row23 = Wire(UInt(16.W))
  when(comp(row10, row13)) {
    row20 := row10            // preserve the first and the forth elements
    row23 := row13
  }.otherwise {
    row20 := row13            // swap the first and the forth elements
    row23 := row10
  }

  when(comp(row20, row21)) {
    io.out0 := row20            // preserve first two elements
    io.out1 := row21
  }.otherwise {
    io.out0 := row21            // swap first two elements
    io.out1 := row20
  }

  when(comp(row22, row23)) {
    io.out2 := row22            // preserve first two elements
    io.out3 := row23
  }.otherwise {
    io.out2 := row23            // swap first two elements
    io.out3 := row22
  }
}
```

ascending是一个参数，它决定了比较函数的行为。

### (3)Option and Default Arguments

```Scala
val map = Map("a" -> 1)
val a = map("a")
println(a)
val b = map("b")
println(b)
```

Map是一个键值对，"a" -> 1是一个键值对，"a"是键，1是值，map("a")是取出键"a"对应的值，map("b")会抛出异常，因为map中没有键"b"。

```Scala
val map = Map("a" -> 1)
val a = map.get("a")
println(a)
val b = map.get("b")
println(b)
```

get()是Map的方法，它返回一个Option，Option是一个抽象类，它有两个子类，Some和None，Some表示有值，None表示没有值，map.get("a")返回Some(1)，map.get("b")返回None。

```Scala
val some = Some(1)
val none = None
println(some.get)          // Returns 1
// println(none.get)       // Errors!
println(some.getOrElse(2)) // Returns 1
println(none.getOrElse(2)) // Returns 2
```

get()是Option的方法，它返回Option的值，如果Option是Some，返回Some的值，如果Option是None，抛出异常。getOrElse()是Option的方法，它返回Option的值，如果Option是Some，返回Some的值，如果Option是None，返回参数的值。

```Scala
class DelayBy1(resetValue: Option[UInt] = None) extends Module {
    val io = IO(new Bundle {
        val in  = Input( UInt(16.W))
        val out = Output(UInt(16.W))
    })
    val reg = if (resetValue.isDefined) { // resetValue = Some(number)
        RegInit(resetValue.get)
    } else { //resetValue = None
        Reg(UInt())
    }
    reg := io.in
    io.out := reg
}

println(getVerilog(new DelayBy1))
println(getVerilog(new DelayBy1(Some(3.U))))
```

resetValue是一个Option，如果resetValue是Some，就用Some的值初始化寄存器，如果resetValue是None，就用Reg()初始化寄存器。

### (4)Match/Case Statements

```Scala
// y is an integer variable defined somewhere else in the code
val y = 7
/// ...
val x = y match {
  case 0 => "zero" // One common syntax, preferred if fits in one line
  case 1 =>        // Another common syntax, preferred if does not fit in one line.
      "one"        // Note the code block continues until the next case
  case 2 => {      // Another syntax, but curly braces are not required
      "two"
  }
  case _ => "many" // _ is a wildcard that matches all values
}
println("y is " + x)
```

match/case语句是Scala的语法，它类似于C语言的switch/case语句。第一种语法是一行的情况常见，第二种语法是多行的情况，第三种语法是多行的情况，但是花括号是可选的，_是通配符，匹配所有值。

在=>右边的代码块执行到结束的花括号或者下一个case语句停止。

match/case语句可以用来匹配多值和类型。

```Scala
def animalType(biggerThanBreadBox: Boolean, meanAsCanBe: Boolean): String = {
  (biggerThanBreadBox, meanAsCanBe) match {
    case (true, true) => "wolverine"
    case (true, false) => "elephant"
    case (false, true) => "shrew"
    case (false, false) => "puppy"
  }
}
println(animalType(true, true))

val sequence = Seq("a", 1, 0.0)
sequence.foreach { x =>
  x match {
    case s: String => println(s"$x is a String")
    case s: Int    => println(s"$x is an Int")
    case s: Double => println(s"$x is a Double")
    case _ => println(s"$x is an unknown type!")
  }
}

val sequence = Seq("a", 1, 0.0)
sequence.foreach { x =>
  x match {
    case _: Int | _: Double => println(s"$x is a number!")//_是占位符
    case _ => println(s"$x is an unknown type!")
  }
}

```

```Scala
class DelayBy1(resetValue: Option[UInt] = None) extends Module {
  val io = IO(new Bundle {
    val in  = Input( UInt(16.W))
    val out = Output(UInt(16.W))
  })
  val reg = resetValue match {
    case Some(r) => RegInit(r)
    case None    => Reg(UInt())
  }
  reg := io.in
  io.out := reg
}

println(getVerilog(new DelayBy1))
println(getVerilog(new DelayBy1(Some(3.U))))
```

### (5)IOs with Optional Fields

```Scala
class HalfFullAdder(val hasCarry: Boolean) extends Module {
  val io = IO(new Bundle {
    val a = Input(UInt(1.W))
    val b = Input(UInt(1.W))
    val carryIn = if (hasCarry) Some(Input(UInt(1.W))) else None
    val s = Output(UInt(1.W))
    val carryOut = Output(UInt(1.W))
  })
  val sum = io.a +& io.b +& io.carryIn.getOrElse(0.U)
  io.s := sum(0)
  io.carryOut := sum(1)
}
```

上面的例子中，carryIn是一个可选的输入，如果hasCarry为true，carryIn是Some，如果hasCarry为false，carryIn是None，getOrElse()是Option的方法，如果Option是Some，返回Some的值，如果Option是None，返回参数的值。

```Scala
class HalfFullAdder(val hasCarry: Boolean) extends Module {
  val io = IO(new Bundle {
    val a = Input(UInt(1.W))
    val b = Input(UInt(1.W))
    val carryIn = Input(if (hasCarry) UInt(1.W) else UInt(0.W))
    val s = Output(UInt(1.W))
    val carryOut = Output(UInt(1.W))
  })
  val sum = io.a +& io.b +& io.carryIn
  io.s := sum(0)
  io.carryOut := sum(1)
}
println("Half Adder:")
println(getVerilog(new HalfFullAdder(false)))
println("\n\nFull Adder:")
println(getVerilog(new HalfFullAdder(true)))
```

上面的例子中，carryIn是一个输入，如果hasCarry为true，carryIn是UInt(1.W)，如果hasCarry为false，carryIn是UInt(0.W)，0位宽的线会被verilog忽略，用到0位宽的值的线会得到常数0。

### (6)Implicits

写log

```scala
sealed trait Verbosity
implicit case object Silent extends Verbosity
case object Verbose extends Verbosity

class ParameterizedWidthAdder(in0Width: Int, in1Width: Int, sumWidth: Int)(implicit verbosity: Verbosity)
extends Module {
  def log(msg: => String): Unit = verbosity match {
    case Silent =>
    case Verbose => println(msg)
  }
  require(in0Width >= 0)
  log(s"in0Width of $in0Width OK")
  require(in1Width >= 0)
  log(s"in1Width of $in1Width OK")
  require(sumWidth >= 0)
  log(s"sumWidth of $sumWidth OK")
  val io = IO(new Bundle {
    val in0 = Input(UInt(in0Width.W))
    val in1 = Input(UInt(in1Width.W))
    val sum = Output(UInt(sumWidth.W))
  })
  log("Made IO")
  io.sum := io.in0 + io.in1
  log("Assigned output")
}

println(getVerilog(new ParameterizedWidthAdder(1, 4, 5)))
println(getVerilog(new ParameterizedWidthAdder(1, 4, 5)(Verbose)))
```

这是一个带有verbosity参数的加法器，verbosity是一个Verbosity类型的隐式参数。

## 3.2 Object Oriented Programming

Scala和Chisel是一门面向对象的语言，拥有java的面向对象的特性。

### (1)Abstract Classes

抽象类没什么特别的，抽象类可以定义许多子类必须实现的未实现值，要注意的是，任何对象只能直接从一个父抽象类继承。

```Scala
abstract class MyAbstractClass {
  def myFunction(i: Int): Int
  val myValue: String
}
class ConcreteClass extends MyAbstractClass {
  def myFunction(i: Int): Int = i + 1
  val myValue = "Hello World!"
}
// Uncomment below to test!
// val abstractClass = new MyAbstractClass() // Illegal! Cannot instantiate an abstract class
val concreteClass = new ConcreteClass()      // Legal!
```

### (2)Traits

特质很像抽象类，但有两点不同：1.一个类可以继承多个特质。2.特质不能有构造函数参数，这意味着特质不能实例化。

```Scala
trait HasFunction {
  def myFunction(i: Int): Int
}
trait HasValue {
  val myValue: String
  val myOtherValue = 100
}
class MyClass extends HasFunction with HasValue {
  override def myFunction(i: Int): Int = i + 1
  val myValue = "Hello World!"
}
// Uncomment below to test!
// val myTraitFunction = new HasFunction() // Illegal! Cannot instantiate a trait
// val myTraitValue = new HasValue()       // Illegal! Cannot instantiate a trait
val myClass = new MyClass()                // Legal!
```

```Scala
class MyClass extends HasTrait1 with HasTrait2 with HasTrait3
```

上面是一个类继承多个特质的例子。

### (3)Objects

Scala为单例类提供了一个称为Objects的语言特性。不能实例化一个Objects(不需要调用new); 可以直接引用它。这使它们类似于Java静态类。

```Scala
object MyObject {
  def hi: String = "Hello World!"
  def apply(msg: String) = msg
}
println(MyObject.hi)
println(MyObject("This message is important!")) // equivalent to MyObject.apply(msg)
```

在上面的例子中，MyObject是一个单例类，hi是一个静态方法，apply是一个静态方法，apply方法可以省略，直接用对象名加括号调用。

### (4)Companion Objects

当一个类和一个对象共享相同的名称并在同一个文件中定义时，该对象称为伴随对象。在class/object名称之前使用new时，它将实例化类。如果不使用new，它将引用对象:

```Scala
object Lion {
    def roar(): Unit = println("I'M AN OBJECT!")
}
class Lion {
    def roar(): Unit = println("I'M A CLASS!")
}
new Lion().roar()
Lion.roar()
```

上面的例子中，Lion是一个伴随对象，Lion是一个类，new Lion()是实例化Lion类，Lion.roar()是调用Lion对象的roar方法。

伴随对象有三个目的：

1. 包含关于类的常数。
2. 在类的构造器前后执行代码。
3. 为类创造多个构造器。

```Scala
object Animal {
    val defaultName = "Bigfoot"
    private var numberOfAnimals = 0
    def apply(name: String): Animal = {
        numberOfAnimals += 1
        new Animal(name, numberOfAnimals)
    }
    def apply(): Animal = apply(defaultName)
}
class Animal(name: String, order: Int) {
  def info: String = s"Hi my name is $name, and I'm $order in line!"
}

val bunny = Animal.apply("Hopper") // Calls the Animal factory method
println(bunny.info)
val cat = Animal("Whiskers")       // Calls the Animal factory method
println(cat.info)
val yeti = Animal()                // Calls the Animal factory method
println(yeti.info)
```

在上面的例子中，Animal是一个伴随对象，Animal定义了两个apply方法，这两个方法都是Animal的工厂方法，因为它们都返回Animal类的实例。

第二个apply方法调用第一个apply方法。

工厂方法(通常通过伴随对象提供)允许以其他方式表示实例创建，为构造函数参数、转换提供额外的测试，并且不需要使用关键字new。

Chisel中有很多伴随对象，例如：

```Scala
val myModule = Module(new MyModule)
```

Module是一个伴随对象，所以Chisel可以在实例化MyModule之前和之后运行后台代码。

### (5)Case Classes

case类是一种特殊的类，它们具有以下特性：
1. 允许外部访问类参数
2. 实例化类时不需要使用new
3. 自动创建一个unapply方法，该方法提供对所有类参数的访问。
4. 不能从子类继承

```Scala
class Nail(length: Int) // Regular class
val nail = new Nail(10) // Requires the `new` keyword
// println(nail.length) // Illegal! Class constructor parameters are not by default externally visible

class Screw(val threadSpace: Int) // By using the `val` keyword, threadSpace is now externally visible
val screw = new Screw(2)          // Requires the `new` keyword
println(screw.threadSpace)

case class Staple(isClosed: Boolean) // Case class constructor parameters are, by default, externally visible
val staple = Staple(false)           // No `new` keyword required
println(staple.isClosed)
```

在声明case类时不需要使用new，这是因为Scala编译器会自动为代码中的每个case类创建一个伴随对象，其中包含case类的apply方法。

### (6)Inheritance with Chisel

每一个Chisel模块都是一个继承了基本类型Module的类。每一个Chisel IO都是一个继承了基本类型Bundle的类。Chisel的硬件类型，如UInt或Bundle，都有Data作为父类型。


