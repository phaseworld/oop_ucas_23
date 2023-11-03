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