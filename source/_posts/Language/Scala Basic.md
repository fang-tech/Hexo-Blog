---
title: Scala基础语法
tags:
  - language
  - scala
---
# Scala运行需要配置
Scala是一门运行在JVM上的语言, 和Java相同会被编译成字节码文件, 也是出于这样的机制, Scala能无缝调用Java的原生基础库, 例如IO, 集合等.

- scalac: 编译器
- scala, scala-cli: CLI工具和工具包(能通过scala run来编译 + 执行, 也可以通过scala命令进入到命令行交互模式)
- sbt: scala build tool
- scalafmt: scala formatter

# Scala的语法基础
## 基础中的基础
- val声明不可变对象, var声明可变对象
- 类型只有"类", 不存在基础类型, 有自动类型推断, 是强类型
- 函数通过 `func(args: Type) => ...`语法声明, 可以匿名
- 方法允许多参数列表
- 函数和方法最后一个表达式的的值就是返回值
- class通过new实例化, object相当于单例类, 会直接初始化生成
- trait相当于接口, case class是VO对象, 能直接实例化不用new关键字

```scala
// object实际上是单例模式
object Basic {

  def main (args: Array[String]) : Unit = {
    val x = 1 + 1;
    var y = 2;
    y = 3;

    // 函数
    val addFunc = (x: Int, y: Int) => x + y
    println(addFunc(1,3))
    // 复杂函数
    val power = (x: Int, p: Int) => {
      var ret = 1;
      for (i <- 0 until p)
        ret *= x
      ret // 最后的表达式的值就是返回值
    }

    println(power(3, 3))
    // 匿名函数
    println((() => 32)()) // 32, 因为调用了这个匿名函数
    println(() => 32) // Basic$$$Lambda$6/1146848448@61a52fbd, 这个函数类的地址

    // 方法
    def testMethod (x: Int, y: Int)(m: Int) : Int = (x + y) * m
    println(testMethod(1,2)(3))

    // 实例化一个类
    val greeter = new Greeter("fang", 23)
    greeter.greet("simon")

    // case class可以不使用new实例化
    val name = FullName("David", "Dai")
    val anothorName = FullName("ff", "fang")
    // 并且能直接比较
    if (name == anothorName)
      println(s"same name $name and $anothorName")
    else
      println(s"different name $name and $anothorName")
  }
}

// 正常的class对象
class Greeter(firstName: String, age: Int) extends BasicGreeter{
  override def greet(lastName: String): Unit =
      println(firstName + lastName + " age is" + age)
}


// 实际上就是VO对象
case class FullName(firstName: String, lastName: String)

trait BasicGreeter {
  def greet(name: String): Unit =
    println(s"Hello $name!!!")
}

// result:
4
27
32
Basic$$$Lambda$6/1146848448@61a52fbd
9
fangsimon age is23
different name FullName(David,Dai) and FullName(ff,fang)
```

## 