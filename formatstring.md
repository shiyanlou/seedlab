##格式化字符串漏洞实验
***

### 一、 实验描述

格式化字符串漏洞是由像printf(user_input)这样的代码引起的，其中user_input是用户输入的数据，具有Set-UID root权限的这类程序在运行的时候，printf语句将会变得非常危险，因为它可能会导致下面的结果：

+ 使得程序崩溃
+ 任意一块内存读取数据
+ 修改任意一块内存里的数据

最后一种结果是非常危险的，因为它允许用户修改set-UID root程序内部变量的值，从而改变这些程序的行为。

本实验将会提供一个具有格式化漏洞的程序，我们将制定一个计划来探索这些漏洞。

###二、实验预备知识讲解

####2.1 什么是格式化字符串？

	printf ("The magic number is: %d", 1911);

试观察运行以上语句，会发现字符串"The magic number is: %d"中的格式符％d被参数（1911）替换，因此输出变成了“The magic number is: 1911”。
格式化字符串大致就是这么一回事啦。

除了表示十进制数的％d，还有不少其他形式的格式符，一起来认识一下吧~

	
格式符| 		含义 | 	含义（英）|传
---- | ------------- | ----- | -------
%d	 | 十进制数（int）  |decimal |  值
%u   | 无符号十进制数 (unsigned int)  | unsigned decimal | 值
%x   | 十六进制数 (unsigned int)  |hexadecimal |  值
%s   | 字符串 ((const) (unsigned) char *)  | string | 引用（指针）
%n   | %n符号以前输入的字符数量 (* int) |number of bytes written so far |  引用（指针）

（ * **%n**的使用将在1.5节中做出说明）

####2.2 栈与格式化字符串
格式化函数的行为由格式化字符串控制，printf函数从栈上取得参数。

	printf ("a has value %d, b has value %d, c is at address: %08x\n",a, b, &c); 

![](http://i.imgur.com/gdtpVzX.png)

####2.3 如果参数数量不匹配会发生什么？
如果只有一个不匹配会发生什么？
```
printf ("a has value %d, b has value %d, c is at address: %08x\n",a, b);
```
+	在上面的例子中格式字符串需要3个参数，但程序只提供了2个。
+	该程序能够通过编译么？
	+	printf()是一个参数长度可变函数。因此，仅仅看参数数量是看不出问题的。
	+	为了查出不匹配，编译器需要了解printf()的运行机制，然而编译器通常不做这类分析。
	+	有些时候，格式字符串并不是一个常量字符串，它在程序运行期间生成(比如用户输入)，因此，编译器无法发现不匹配。
+	那么printf()函数自身能检测到不匹配么？
	+	printf()从栈上取得参数，如果格式字符串需要3个参数，它会从栈上取3个，除非栈被标记了边界，printf()并不知道自己是否会用完提供的所有参数。
	+	既然没有那样的边界标记。printf()会持续从栈上抓取数据，在一个参数数量不匹配的例子中，它会抓取到一些不属于该函数调用到的数据。
+ 如果有人特意准备数据让printf抓取会发生什么呢？

#### 2.4 访问任意位置内存
+ 我们需要得到一段数据的内存地址，但我们无法修改代码，供我们使用的只有格式字符串。
+ 如果我们调用 printf(%s) 时没有指明内存地址, 那么目标地址就可以通过printf函数，在栈上的任意位置获取。printf函数维护一个初始栈指针,所以能够得到所有参数在栈中的位置
+ 观察: 格式字符串位于栈上. 如果我们可以把目标地址编码进格式字符串，那样目标地址也会存在于栈上，在接下来的例子里，格式字符串将保存在栈上的缓冲区中。

```
int main(int argc, char *argv[])
{
	char user_input[100];
	... ... /* other variable definitions and statements */
	scanf("%s", user_input); /* getting a string from user */
	printf(user_input); /* Vulnerable place */
	return 0;
}
```

+ 如果我们让printf函数得到格式字符串中的目标内存地址 (该地址也存在于栈上), 我们就可以访问该地址.

		printf ("\x10\x01\x48\x08 %x %x %x %x %s");

+ \x10\x01\x48\x08 是目标地址的四个字节， 在C语言中, \x10 告诉编译器将一个16进制数0x10放于当前位置（占1字节）。如果去掉前缀\x10就相当于两个ascii字符1和0了，这就不是我们所期望的结果了。
+ %x 导致栈指针向格式字符串的方向移动（参考1.2节）
+ 下图解释了攻击方式，如果用户输入中包含了以下格式字符串
![](http://i.imgur.com/gHmSCX6.png)
+ 如图所示，我们使用四个%x来移动printf函数的栈指针到我们存储格式字符串的位置，一旦到了目标位置，我们使用％s来打印，它会打印位于地址0x10014808的内容，因为是将其作为字符串来处理，所以会一直打印到结束符为止。
+ user_input数组到传给printf函数参数的地址之间的栈空间不是为了printf函数准备的。但是，因为程序本身存在格式字符串漏洞，所以printf会把这段内存当作传入的参数来匹配％x。
+ 最大的挑战就是想方设法找出printf函数栈指针(函数取参地址)到user_input数组的这一段距离是多少，这段距离决定了你需要在%s之前输入多少个%x。

####2.5 在内存中写一个数字
%n: 该符号前输入的字符数量会被存储到对应的参数中去

	int i;
	printf ("12345%n", &i);
	
+ 数字5（%n前的字符数量）将会被写入i 中
+ 运用同样的方法在访问任意地址内存的时候，我们可以将一个数字写入指定的内存中。只要将上一小节(1.4)的%s替换成%n就能够覆盖0x10014808的内容。
+ 利用这个方法，攻击者可以做以下事情:
	+ 重写程序标识控制访问权限
	+ 重写栈或者函数等等的返回地址
+ 然而，写入的值是由%n之前的字符数量决定的。真的有办法能够写入任意数值么？
	+ 用最古老的计数方式， 为了写1000，就填充1000个字符吧。
	+ 为了防止过长的格式字符串，我们可以使用一个宽度指定的格式指示器。(比如（%0数字x）就会左填充预期数量的0符号)

### 三、 实验内容
用户需要输入一段数据，数据保存在user_input数组中，程序会使用printf函数打印数据内容，并且该程序以root权限运行。更加可喜的是，这个程序存在一个格式化漏洞。让我们来看看利用这些漏洞可以搞些什么破坏。

程序说明：
>程序内存中存在两个秘密值，我们想要知道这两个值，但发现无法通过读二进制代码的方式来获取它们（实验中为了简单起见，硬编码这些秘密值为0x44和0x55）。尽管我们不知道它们的值，但要得到它们的内存地址倒不是特别困难，因为对大多数系统而言，每次运行程序，这些内存地址基本上是不变的。实验假设我们已经知道了这些内存地址，为了达到这个目的，程序特意为我们打出了这些地址。
>
>有了这些前提以后我们需要达到以下目标：
>+	找出secret[1]的值
>+	修改secret[1]的值
>+	修改secret[1]为期望值

>**注意**：因为实验环境是64位系统，所以需要使用%016llx才能读取整个字。但为了简便起见，对程序进行了修改了，使用%08x也能完成实验。

有了之前预备知识的铺垫，先自己尝试一下，祝玩的愉快：）

程序如下：
```
/* vul_prog.c */ 
#include <stdlib.h>
#include <stdio.h>

#define SECRET1 0x44
#define SECRET2 0x55

int main(int argc, char *argv[])
{
  char user_input[100];
  int *secret;
  long int_input;
  int a, b, c, d; /* other variables, not used here.*/

  /* The secret value is stored on the heap */
  secret = (int *) malloc(2*sizeof(int));

  /* getting the secret */
  secret[0] = SECRET1; secret[1] = SECRET2;

  printf("The variable secret's address is 0x%8x (on stack)\n", &secret);
  printf("The variable secret's value is 0x%8x (on heap)\n", secret);
  printf("secret[0]'s address is 0x%8x (on heap)\n", &secret[0]);
  printf("secret[1]'s address is 0x%8x (on heap)\n", &secret[1]);

  printf("Please enter a decimal integer\n");
  scanf("%d", &int_input);  /* getting an input from user */
  printf("Please enter a string\n");
  scanf("%s", user_input); /* getting a string from user */

  /* Vulnerable place */
  printf(user_input);  
  printf("\n");

  /* Verify whether your attack is successful */
  printf("The original secrets: 0x%x -- 0x%x\n", SECRET1, SECRET2);
  printf("The new secrets:      0x%x -- 0x%x\n", secret[0], secret[1]);
  return 0;
}
```
(ps： 编译时可以添加以下参数关掉栈保护。)

	gcc -z execstack -fno-stack-protector -o vul_prog vul_prog.c 

---

一点小提示：你会发现secret[0]和secret[1]存在于malloc出的堆上，我们也知道secret的值存在于栈上，如果你想覆盖secret[0]的值,ok,它的地址就在栈上，你完全可以利用格式化字符串的漏洞来达到目的。然而尽管secret[1]就在它的兄弟0的旁边，你还是没办法从栈上获得它的地址，这对你来说构成了一个挑战，因为没有它的地址你怎么利用格式字符串读写呢。但是真的就没招了么？

---

####3.1 找出secret[1]的值
1.首先定位int_input的位置，这样就确认了％s在格式字符串中的位置。
![](http://i.imgur.com/FwK544g.png)

2.输入secret[1]的地址，记得做进制转换，同时在格式字符串中加入％s。
![](http://i.imgur.com/NLD7k8e.png)

大功告成！U的ascii码就是55。

####3.2 修改secret[1]的值
1.只要求修改，不要求改什么？简单！不明白%n用法的可以往前回顾一下。
![](http://i.imgur.com/mEDs0NO.png)

大功告成x2！

####3.3 修改secret[1]为期望值

1.要改成自己期望的值，咋办？填1000岂不累死？！可以用填充嘛！
![](http://i.imgur.com/EAy5uaS.png)
哦对了，0x3e8 = 1000。
大功告成x3！

###四、 练习

在实验楼环境安步骤进行实验，并截图

您已经完成本课程的所有实验，**干的漂亮！**

### License

本课程所涉及的实验来自[Syracuse SEED labs](http://www.cis.syr.edu/~wedu/seed/)，并在此基础上为适配[实验楼](http://www.shiyanlou.com)网站环境进行修改，修改后的实验文档仍然遵循GNU Free Documentation License。

本课程文档github链接：[https://github.com/shiyanlou/seedlab](https://github.com/shiyanlou/seedlab)

附[Syracuse SEED labs](http://www.cis.syr.edu/~wedu/seed/)版权声明：

> Copyright Statement Copyright 2006 – 2009 Wenliang Du, Syracuse University. The development of this document is funded by the National Science Foundation’s Course, Curriculum, and Laboratory Improvement (CCLI) program under Award No. 0618680 and 0231122. Permission is granted to copy, distribute and/or modify this document under the terms of the GNU Free Documentation License, Version 1.2 or any later version published by the Free Software Foundation. A copy of the license can befound at http://www.gnu.org/licenses/fdl.html.
