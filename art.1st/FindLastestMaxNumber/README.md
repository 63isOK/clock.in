# 这是一个寻找最后一个最大数的算法

    * max.mixal : find lastest max number fo a array
    *
    * algorithm M
    * 1.[Initialize.] k=n,j=n,m=X[n],k=n-1.
    * 2.[All tested?] return; if k>0, To M3.
    * 3.[Compare.] if m >= X[k], To M4. others, To M5.
    * 4.[Change m.] j=k,m=X[k],To M5.
    * 5.[Decrease k.] k=k-1. To M2.
    *
    * label   ins   operand     comment
    X         EQU   1000        address of array
      | | | | ORIG  3000
    MAXNUM    STJ   EXIT
    INIT      ENT3  0,1
      | | | | JMP   CHANGEM
    LOOP      CMPA  X,3
      | | | | JGE   *+3
    CHANGEM   ENT2  0,3
      | | | | LDA   X,3
      | | | | DEC3  1
      | | | | JMP   LOOP
    EXIT      JMP   *

我们来具体解读一下这个例子.

算法过程很简单:

- 标记最后一个数为最大数
- 从后往前遍历,找出比标记数还大的数,替换标记
- 直到,第一个数也被遍历完
- 那个标记的数就是最后最大数

现在看看源码部分:

- 行头,"标签-操作码-操作数-注释",原则上是要写的
- 使用mixal写的程序,需要经过mixasm翻译成机器能懂的语言
  - 所以对于使用mixal的编码者,无需关心实际的数字代码
- 行号,不是必须的,仅仅是增加了示例的可读性
- 注释是来解释算法的,而我们的编码仅仅是将算法翻译成mixal
- 耗时是体现每个指令执行的次数,这个是算法研究的重点

下面用段落的方式来mixal源码:

X EQU 1000, 定义了一个符号X,表示1000.这里表示X数组的起始.

ORIG 3000, 表示程序启动之后,将pc置为3000.

MAXNUM STJ EXIT, 这是启动之后执行的第一条语句,
MAXNUM的值就是3000,下面的指令地址依次加1.

EQU/ORIG和下面的STJ等指令是略有差别的,
EQU/ORIG是伪指令,因为他们是mixal的指令,而不是mix的指令,
说白了,mixal的指令会在mixasm中进行特殊转换的,
伪指令更多的是告诉mixasm一些信息,而不是告诉mixasm具体指令.

再来看MAXNUM STJ EXIT,MAXNUM是3000,STJ是将一个地址保存到rJ中,
