Macros 的一个简单入门教程
### Macros 何去何从
Scala Macro 是 Scala 2.10 版本增加的一个新的语言特性，让开发者可以在编译期动态修改/生成代码，为开发工作提供了很大的灵活性。到了Scala 2.12，Scala Macro 基本上没有太多变化，并且直到目前为止，Scala Macro 一直被打着EXPERIMENTAL标签。据scala官网介绍，scala3的目前披露的计划中，scala Macros 可能会被官方抛弃，取而代之的是 Dotty 中的重新实现 。
有兴趣的同学可以参考：https://www.scala-lang.org/blog/2018/04/30/in-a-nutshell.html 学习为何要被抛弃替代的原因：
1.  macro 的实现完全依赖于当前的scala编译器(nsc)， 但scala3.0 将使用全新的scala编译器(dotc) 
2.  macro 的实现缺少基础语法，与scala3.0的规范不一致，将会使用Principled Meta Programming
 进行替代相关语法实现。 http://dotty.epfl.ch/docs/reference/principled-meta-programming.html 


基于以上两个关键原因，scala macros  将被重构至 Tasty 中。 
 
### Macros 爱恨交加
优点: 显著简化了代码分析和代码生成，这使得它们成为处理大量现实用例的一种可选工具。传统上涉及编写和维护样板的场合可用宏以简单且易维护的方式实现。比如我们常见的实现情景：  jsonPaser 在json的处理上， 使用macro 通过生成样板代码实现，并且效率较高
- 缺点: 
1. 编写复杂
2. 调试麻烦
3. 编译速度变慢
4. 编译文件变大

- 优点：
1. 可以用于优化某些特定场景，例如 json转换和resultSet转换

### Talk is cheap. Show me the code
#### HelloWorld
LibraryMacros.scala
```scala
import java.util.Date

import scala.reflect.macros.blackbox.Context
import scala.language.experimental.macros

object LibraryMacros {
  def greeting: String = macro greetingMacro

  def greetingMacro(c: Context): c.Tree = {
    import c.universe._

    val now = new Date().toString

    q"""
     "Hi! This code was compiled at " +
     $now
     """
  }
}

```

HelloMacros.scala
```scala
object HelloMacros extends App {
  import LibraryMacros._

  
  /**
  * 执行结果将会是 : Hi! This code was compiled at ${HelloMacros的编译时间}
  * 
  * 编译的过程如下：
  *   当编译器在编遇到方法调用greeting时会进行函数符号解析,在LibraryMacros 里发现greeting是个macro，它的具体实现在greetingMacro函数里.
  *   此时编译器会运行greetingMacro函数并将运算结果-一个Scala的抽象语法树(AST) ， 编译器会就着新插入的 AST 继续执行编译，达到在编译期动态修改/生成代码 的目的
  * 
  * 经过反编译得到的结果如下：
  *     public final void delayedEndpoint$HelloMacros$1() {
  *         Predef$.MODULE$.println("Hi! This code was compiled at Wed Jul 04 15:49:40 CST 2018");
  *     }
  */
  println(greeting)

}

```

#### 教程分享
大家可以参考https://github.com/underscoreio/essential-macros
由浅入深，从各个方面介绍了如何进行一步步的macro编程步骤，建议大家按照以下顺序进行阅读与尝试
- hello - 在打印的消息中嵌入了一个编译时间戳，以证明它是在编译时执行的。 这与在运行时打印时间戳的常规代码形成对比。

- maximum - 简单的项目演示基本设置。 最大的宏本身并不是特别有用，但该项目是各种概念的一个很好的例子。

- printtree - 使用showCode和showRaw的宏来打印Scala代码的任意片段的desugared语法和底层树结构。 有利于在编写Macro程序之前进行研究。

- simpleassert - Macros宏使用quasiquotes展示在树上的简单模式匹配。 稍微改进的Scala内置断言方法版本，可以打印断言中涉及的各种值。

- betterassert - 使用模式匹配和树遍历演示更高级的树检查。 simpleassert的改进版本，可在更广泛的案例中打印有用的调试信息。

- printtype - 通用Macros系列，可以打印有关其类型参数的各种信息。有利于在编写Macro程序期间进行调试与研究

- orderings - 代码生成Macros，用于检查类型并创建允许按任何字段排序的对象。 例如，在编写允许用户通过返回数据中的任何字段对数据库进行排序的Web服务时，此技术非常有用。

- enumerations - 生成一系列的 sealed trait, 用于创建你自己的枚举对象 

- whitebox - 简单的项目展示了whitebox和blackbox宏之间的根本区别。

- validation - 验证库的代码设计工具，使用错误字段的名称自动标记错误。 一个重要的属性是Macros被实现为可链接的方法调用而不是顶级函数。

- csv - 用于CSV序列化的基于类型的代码库的设计工具。 隐式Macros用于支持案例类的类型类实例的自动实现。


#### scala-sql 对于macro的应用
scala-sql中 对于macro的应用分为两块
1. 第一块为 ResultSetMapper 的生成，通过生成代码，解决了 每个bean都需要一个ResultSetMapper[T]的痛点
2. BeanBuilder 的转换工作，解决了 case class 的转换

