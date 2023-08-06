# 2.java怒你急编译器
&emsp;&emsp;JVM机器旨在支持 Java 编程语言。 Oracle 的 JDK 软件包含一个由 Java 编程语言编写的将源代码编译成 Java 虚
拟机指令集的编译器，以及一个实现 Java 虚拟机本身的运行时系统。 理解一个编译器如何利用 Java 虚拟机对未来
的编译器作者以及试图理解 Java 虚拟机本身的人来说都是有用的。 本章中内容不是规范性的。

&emsp;&emsp;请注意，当指代从 Java 虚拟机的指令集到特定 CPU 的指令集的翻译器时，有时会使用术语“编译器”。 这种翻译器
的一个示例是即时 (JIT) 代码生成器，它仅在加载 Java 虚拟机代码后生成特定于平台的指令。 本章不解决与代码生
成相关的问题，仅解决与将用 Java 编程语言编写的源代码编译为 Java 虚拟机指令相关的问题。
# 2.1. 示例的格式
&emsp;&emsp;本章主要包含源代码示例以及 Oracle 的 JDK 版本 1.0.2 中的 javac 编译器为示例生成的 Java 虚拟机代码的注解列表。 Java 虚拟机代码是用 Oracle 的 javap 实用程序输出的非正式“虚拟机汇编语言”编写的，随 JDK 版本一起分发。 您可以使用 javap 生成已编译方法的其他示例。
阅读过汇编代码的人都应该熟悉这些示例的格式。 每条指令采用以下形式：

````
<index> <opcode> [ <operand1> [ <operand2>... ]] [<comment>]
````
* <index> 是数组中指令操作码的索引，该数组包含此方法的 Java 虚拟机代码字节。 
或者，可以将 <index> 视为距方法开头的字节偏移量。
* <opcode>是指令操作码的助记符，零个或多个<operandN>是指令的操作数。
可选的 <comment> 以行尾注解语法给出：
````
bipush 100     // Push int constant 100
````
&emsp;&emsp;每条指令前面的 <index> 可以用作控制传输指令的目标。 例如，goto 8指令将控制转移到索引 8 处的指令。请注意，Java 虚
拟机控制转移指令的实际操作数是这些指令操作码地址的偏移量； 这些操作数由 javap 显示（并在本章中显示），因为更容易将偏移量读取
到它们的方法中。
我们在表示运行时常量池索引的操作数前加上井号，并在指令后面加上标识所引用的运行时常量池项的注解，如下所示：
````
less复制代码10  ldc #1         // Push float constant 100.0
````
或
````
invokevirtual #4    // Method Example.addTwo(II)I
````
## 2.2. 常量、局部变量和控制结构
&emsp;&emsp;Java 虚拟机代码展示了一组由 Java 虚拟机的设计和类型使用强加的一般规范。 在第一个例子中，我们遇到了很多这样的情况，
我们对它们进行了一些详细的探讨。
spin 方法简单地围绕一个空的 for 循环旋转 100 次：
````
void spin() {
     int i;
     for (i = 0; i < 100; i++) {
          ; // Loop body is empty
     }
}
````
然后编译器编译后
````
0   iconst_0       // Push int constant 0
1   istore_1       // Store into local variable 1 (i=0)
2   goto 8         // First time through don't increment
5   iinc 1 1       // Increment local variable 1 by 1 (i++)
8   iload_1        // Push local variable 1 (i)
9   bipush 100     // Push int constant 100
11  if_icmplt 5    // Compare and loop if less than (i < 100)
14  return         // Return void when done.
````
&emsp;&emsp;Java 虚拟机是面向栈的，大多数操作从 Java 虚拟机当前帧的操作数栈中获取一个或多个操作数，或者将结果推回操作数栈。 每次调用方法时
都会创建一个新帧，并创建一个新的操作数栈和一组供该方法使用的局部变量。 因此，在计算的任何一点，每个控制线程都可能有许
多帧和同样多的操作数栈，对应于许多嵌套方法调用。 只有当前帧中的操作数栈是活动的。
&emsp;&emsp;Java 虚拟机的指令集通过使用不同的字节码对其各种数据类型进行操作来区分操作数类型。 spin 方法仅对 int 类型的值进行操作。 其编译
代码中选择对类型数据（iconst_0、istore_1、iinc、iload_1、if_icmplt）进行操作的指令都是专门针对 int 类型的。
自旋中的两个常量 0 和 100 使用两条不同的指令被压入操作数栈。 使用 iconst_0 指令（iconst_<i> 指令家族之一）推送 0。 使用 bipush 指令推送 100，
该指令获取它推送的值作为立即操作数。
&emsp;&emsp;Java 虚拟机经常利用某些操作数（在 iconst_<i> 指令的情况下为 int 常量 -1、0、1、2、3、4 和 5）的可能性，方法是使这些操作数隐含在操作码中。 
因为 iconst_0 指令知道它要压入一个 int 0，所以 iconst_0 不需要存储操作数来告诉它压入什么值，也不需要获取或解码操作数。 将 0 的推送编译为 bipush 0 是正确的，
但会使自旋的编译代码长一个字节。 一个简单的虚拟机还会在每次循环中花费额外的时间来获取和解码显式操作数。 使用隐式操作数使编译后的代码更加紧凑和高效。
spin 中的 int i 存储为 Java 虚拟机局部变量 1。由于大多数 Java 虚拟机指令对从操作数栈弹出的值进行操作，而不是直接对局部变量进行操作，因此在局部变量和操作数栈
之间传递值的指令在 为 Java 虚拟机编译的代码。 这些操作在指令集中也有特殊的支持。 在自旋中，使用 istore_1 和 iload_1 指令将值传入和传出局部变量，每条指令都隐
式地对局部变量 1 进行操作。istore_1 指令从操作数栈中弹出一个 int 并将其存储在局部变量 1 中。iload_1 指令压入 将局部变量 1 中的值压入操作数栈。
&emsp;&emsp;局部变量的使用（和重用）是编译器作者的责任。 专门的加载和存储指令应该鼓励编译器编写者尽可能多地重用局部变量。 生成的代码更快、更紧凑，并且在栈帧中使用的空间更少。
Java 虚拟机专门针对局部变量进行的某些非常频繁的操作。 iinc 指令将局部变量的内容递增一个字节的有符号值。 自旋中的 iinc 指令将第一个局部变量（它的第一个操作数）递增 1（它的第二个操作数）。 iinc 指令在实现循环结构时非常方便。
spain的for循环主要是通过以下指令完成的：
````
5   iinc 1 1       // Increment local variable 1 by 1 (i++)
8   iload_1        // Push local variable 1 (i)
9   bipush 100     // Push int constant 100
11  if_icmplt 5    // Compare and loop if less than (i < 100)
````
bipush 指令将值 100 作为 int 压入操作数栈，然后 if_icmplt 指令将该值从操作数栈弹出并将其与 i 进行比较。 如果比较成功（变量 i 小于 100），则控制权转移到索引 5，for 循环的下一次迭代开始。 否则，控制传递给 if_icmplt 之后的指令。
如果自旋示例为循环计数器使用了 int 以外的数据类型，则编译后的代码必然会更改以反映不同的数据类型。 例如，如果自旋示例使用双精度而不是整数，如下所示：
````
void dspin() {
    double i;
    for (i = 0.0; i < 100.0; i++) {
        ;    // Loop body is empty
    }
}
````
编译后的代码为
````
Method void dspin()
0   dconst_0       // Push double constant 0.0
1   dstore_1       // Store into local variables 1 and 2
2   goto 9         // First time through don't increment
5   dload_1        // Push local variables 1 and 2
6   dconst_1       // Push double constant 1.0
7   dadd           // Add; there is no dinc instruction
8   dstore_1       // Store result in local variables 1 and 2
9   dload_1        // Push local variables 1 and 2
10  ldc2_w #4      // Push double constant 100.0
13  dcmpg          // There is no if_dcmplt instruction
14  iflt 5         // Compare and loop if less than (i < 100.0)
17  return         // Return void when done
````
对类型化数据进行操作的指令现在专用于双精度类型。 
回想一下，double 值占用两个局部变量，尽管只能使用两个局部变量中较小的索引访问它们。 对于 long 类型的值也是如此。 再举个例子，
````
double doubleLocals(double d1, double d2) {
    return d1 + d2;
}
````
变成了
````
Method double doubleLocals(double,double)
0   dload_1       // First argument in local variables 1 and 2
1   dload_3       // Second argument in local variables 3 and 4
2   dadd
3   dreturn
````

请注意，用于在 doubleLocals 中存储双精度值的局部变量对的局部变量绝不能单独操作。
Java 虚拟机的 1 字节操作码大小导致其编译后的代码非常紧凑。 然而，1 字节操作码也意味着 Java 虚拟机指令集必须保持较小。 作为一种妥协，Java 虚拟机并没有为所有数据类型提供同等的支持：它不是完全正交的 。
例如spin的for语句中int类型值的比较，可以用一条if_icmplt指令来实现； 但是，在 Java 虚拟机指令集中没有一条指令可以对 double 类型的值执行条件分支。 因此，dspin 必须使用紧跟 iflt 指令的 dcmpg 指令来实现其对 double 类型值的比较。
Java虚拟机对int类型的数据提供了最直接的支持。 这在一定程度上是为了高效实现 Java 虚拟机的操作数栈和局部变量数组。 它也受到典型程序中 int 数据频率的推动。 其他整数类型的直接支持较少。 例如，没有byte、char或shot的store、load或add指令。 这是使用 short 编写的自旋示例：
````
void sspin() {
short i;
for (i = 0; i < 100; i++) {
    ;    // Loop body is empty
    }
}
````
它必须为 Java 虚拟机编译，如下所示，使用对另一种类型（很可能是 int）进行操作的指令，根据需要在 short 和 int 值之间进行转换，以确保对 short 数据的操作结果保持在适当的范围内：
````
Method void sspin()
0   iconst_0
1   istore_1
2   goto 10
5   iload_1        // The short is treated as though an int
6   iconst_1
7   iadd
8   i2s            // Truncate int to short
9   istore_1
10  iload_1
11  bipush 100
13  if_icmplt 5
16  return
````
Java 虚拟机缺乏对 byte、char 和 short 类型的直接支持并不是特别痛苦，因为这些类型的值在内部被提升为 int（byte 和 short 被符号扩展为 int，char 被零扩展） . 因此可以使用 int 指令完成对 byte、char 和 short 数据的操作。 唯一的额外成本是将 int 操作的值截断到有效范围。
长整型和浮点类型在 Java 虚拟机中具有中等水平的支持，只缺少条件控制传输指令的完整补充。
## 1.3. 算术
Java 虚拟机通常在其操作数栈上进行算术运算。 （例外是 iinc 指令，它直接递增局部变量的值。）例如，align2grain 方法将 int 值与给定的 2 的幂对齐：
````
int align2grain(int i, int grain) {
return ((i + grain-1) & ~(grain-1));
}
````
算术运算的操作数从操作数栈弹出，运算结果被推回操作数栈。 因此，算术子计算的结果可以用作它们的嵌套计算的操作数。 例如，~(grain-1) 的计算由这些指令处理：
````
    5   iload_2        // Push grain
    6   iconst_1       // Push int constant 1
    7   isub           // Subtract; push result
    8   iconst_m1      // Push int constant -1
    9   ixor           // Do XOR; push result
````
首先，使用局部变量 2 的内容和立即 int 值 1 计算 grain-1。这些操作数从操作数堆栈中弹出，并将它们的差值推回操作数堆栈。 因此，差异可立即用作 ixor 指令的一个操作数。 （回想一下 ~x == -1^x。）类似地，ixor 指令的结果成为后续 iand 指令的操作数。
完整代码如下
````
Method int align2grain(int,int)
    0   iload_1
    1   iload_2
    2   iadd
    3   iconst_1
    4   isub
    5   iload_2
    6   iconst_1
    7   isub
    8   iconst_m1
    9   ixor
    10  iand
    11  ireturn
````

## 2.4. 访问运行时常量池
许多数字常量，以及对象、字段和方法，都是通过当前类的运行时常量池访问的。  使用 ldc、ldc_w 和 ldc2_w 指令管理 int、long、float 和 double 类型的数据，以及对 String 类实例的引用。
ldc 和 ldc_w 指令用于访问运行时常量池（包括类 String 的实例）中除 double 和 long 之外的类型的值。
ldc_w 指令仅在存在大量运行时常量池项并且需要更大的索引来访问项时才用于代替 ldc。
ldc2_w 指令用于访问所有 double 和 long 类型的值；
可以使用 bipush、sipush 或 iconst_<i> 指令（§3.2）编译 byte、char 或 short 类型的整型常量，以及小的 int 值。 可以使用 fconst_<f> 和 dconst_<d> 指令编译某些小的浮点常量。
在所有这些情况下，编译都很简单。 例如，以下常量：
````
void useManyNumeric() {
    int i = 100;
    int j = 1000000;
    long l1 = 1;
    long l2 = 0xffffffff;
    double d = 2.2;
    ...do some calculations...
}
````

设置如下：
````
Method void useManyNumeric()
    0   bipush 100   // Push small int constant with bipush
    2   istore_1
    3   ldc #1       // Push large int constant (1000000) with ldc
    5   istore_2
    6   lconst_1     // A tiny long value uses small fast lconst_1
    7   lstore_3
    8   ldc2_w #6    // Push long 0xffffffff (that is, an int -1)
    // Any long constant value can be pushed with ldc2_w
    11  lstore 5
    13  ldc2_w #8    // Push double constant 2.200000
    // Uncommon double values are also pushed with ldc2_w
    16  dstore 7
    ...do those calculations...
````
## 2.5. 更多控制示例
 