##SET-UID程序漏洞实验

----

在线实验环境：[SET-UID程序漏洞实验](http://www.shiyanlou.com/courses/227)

###一、实验描述

Set-UID 是Unix系统中的一个重要的安全机制。当一个Set-UID程序运行的时候，它被假设为具有拥有者的权限。例如，如果程序的拥有者是root，那么任何人运行这个程序时都会获得程序拥有者的权限。Set-UID允许我们做许多很有趣的事情，但是不幸的是，它也是很多坏事情的罪魁祸首。

因此本次实验的目标有两点：

> 1．欣赏好的方面，理解为什么Set-UID是需要的，以及它是如何被执行的。

> 2．注意坏的方面，理解它潜在的安全性问题。

###二、实验内容

这是一个探索性的实验，你的任务是在Linux环境中和Set-UID机制”玩游戏“，你需要在Linux中完成接下来的实验任务：

####2.1 猜测为什么“passwd”，“chsh”，“su”，和“sudo”命令需要Set-UID机制，如果它们没有这些机制的话，会发生什么，如果你不熟悉这些程序，你可以通话阅读使用手册来熟悉它们，如果你拷贝这些命令到自己的目录下，这些程序就不会是Set-UID程序，运行这些拷贝的程序，观察将会发生什么。

![tu-01](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-01.png)

从上面的截图可以看出：将passwd拷贝到/tmp/下，权限发生了变化（在原目录下suid位被设置），复件没有了修改密码的权限。

对于“chsh”，“su”，和“sudo”命令，把这些程序拷贝到用户目录下，同样不再具有root权限。

####2.2 在linux环境下运行Set-UID 程序，同时描述并且解释你的观察结果

#####2.2.1 以root方式登录，拷贝/bin/zsh 到/tmp, 同时设置拷贝到tmp目录下的zsh为set-uid root权限，然后以普通用户登录，运行/tmp/zsh。你会得到root权限吗？请描述你的结果。

![tu-02](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-02.png)

#####2.2.2 拷贝/bin/bash到/tmp目录，同时设置/tmp目录下的bash为Set-UID root权限，然后以普通用户登录，运行/tmp/bash。你会得到root权限吗？请描述你的结果。

![tu-03](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-03.png)

可见，同样的操作，运行复制的zsh可以获得root权限，而bash不能。

####2.3 从上面步骤可以看出，/bin/bash有某种内在的保护机制可以阻止Set-UID机制的滥用。为了能够体验这种内在的保护机制出现之前的情形，我们打算使用另外一种shell程序——/bin/zsh。在一些linux的发行版中（比如Redora和Ubuntu），/bin/sh实际上是/bin/bash的符号链接。为了使用zsh，我们需要把/bin/sh链接到/bin/zsh。

下面的指令将会把默认的shell指向zsh：

```
$sudo su
Password:
#cd /bin
#rm sh
#ln -s zsh sh
```

![tu-04](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-04.png)

####2.4 PATH环境变量的设置

system(const char * cmd)系统调用函数被内嵌到一个程序中执行一个命令，system()调用/bin/sh来执行shell程序，然后shell程序去执行cmd命令。但是在一个Set-UID程序中system()函数调用shell是非常危险的，这是因为shell程序的行为可以被环境变量影响，比如PATH；而这些环境变量可以在用户的控制当中。通过控制这些变量，用心险恶的用户就可以控制Set-UID程序的行为。

下面的Set-UID程序被用来执行/bin/ls命令；然后程序员可以为ls命令使用相对路径，而不是绝对路径。

```
int main()
{
   system("ls");
   return 0;
}
```

#####2.4.1 你能够设置这个Set-UID程序运行你自己的代码而不是/bin/ls吗？如果你能的话，你的代码具有root权限吗？描述并解释你的观察。

可以具有root权限，把/bin/sh拷贝到/tmp目录下面重命名为ls（先要确保/bin/目录下的sh 符号链接到zsh，而不是bash），将环境变量PATH设置为当前目录/tmp，运行编译的程序test。就可以获得root权限：

![tu-05](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-05.png)

#####2.4.2 修改/bin/sh使得其返回到/bin/bash，重复上面的攻击，你仍然可以获得root权限吗？描述并解释你的观察。

![tu-06](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-06.png)

可见修改sh连接回bash，运行test程序不能使普通用户获得root权限。

####2.5 sytem()和execve()的不同

首先确保/bin/sh指向zsh

背景：Bob在为一家审计代理处工作，他正在调查一家公司是否存在诈骗行为。为了这个目的，他需要阅读这家公司在Unix系统中的所有文件；另一方面，为了保护系统的可靠性，他不能修改任何一个文件。为了达到这个目的，Vince——系统的超级用户为他写了一个SET-ROOT-UID程序，并且给了Bob可以执行它的权限。这个程序需要Bob在命令行中打出一个文件名，然后运行/bin/cat命令显示这个文件。既然这个程序是以root权限运行的，它就可以显示Bob想看的任何一个文件。然而，既然这个程序没有写操作，Vince很确信Bob不能用这个程序修改任何文件。

```
#include <string.h>
#include <stdio.h>
#include <stdlib.h>
int main(int argc, char *argv[])
{
   char *v[3];
   if(argc < 2)
   {
   printf("Please type a file name.\n");
   return 1;
   }
   v[0] = "/bin/cat"; v[1] = argv[1]; v[2] = 0;
  //Set q = 0 for Question a, and q = 1 for Question b
   int q = 0;
   if (q == 0)
   {
      char *command = malloc(strlen(v[0]) + strlen(v[1]) + 2);
      sprintf(command, "%s %s", v[0], v[1]);
      system(command);
  }
  else execve(v[0], v, 0);
  return 0 ;
}
```

#####2.5.1 程序中有 q=0。程序会使用system()调用命令行。这个命令安全码？如果你是Bob，你能对系统的完整性妥协吗？你能重新移动一个对你没有写权限的文件吗?

这个命令不安全，Bob可能会出于好奇或者个人利益驱使阅读或者修改只有root用户才可以运行的一些文件。比如截图中：file文件只有root用户有读写权限，但普通用户通过运行该程序，阅读并重命名了file文件：

![tu-07](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-07.png)

#####2.5.2 如果令q=1；刚才的攻击还会有效吗？请描述并解释你的观察。

修改为q=1后，不会有效。前面步骤之所以有效，是因为system()函数调用/bin/sh，链接至zsh，具有root权限执行了cat file文件后，接着执行mv file file_new命令。

而当令q=1, execve()函数会把file; mv file file_new 看成是一个文件名，系统会提示不存在这个文件：

![tu-08](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-08.png)

####2.6 LD_PRELOAD环境变量

为了保证Set-UID程序在LD_PRELOAD环境的操纵下是安全的，动态链接器会忽略环境变量，但是在某些条件下是例外的，在下面的任务中，我们猜测这些特殊的条件到底是什么。

1、让我们建立一个动态链接库。把下面的程序命名为mylib.c，放在/tmp目录下。在函数库libc中重载了sleep函数：

```
#include <stdio.h>
void sleep (int s)
{
    printf("I am not sleeping!\n");
}
```

2、我们用下面的命令编译上面的程序（注意区别l和1）：

```
gcc -fPIC -g -c mylib.c

gcc -shared -Wl,-soname,libmylib.so.1 \
-o libmylib.so.1.0.1 mylib.o –lc
```

3、把下面的程序命名为myprog.c，放在/tmp目录下：

```
int main()
{
   sleep(1);
   return 0;
}
```

请在下面的条件下运行这些程序，并观察结果。基于这些观察告诉我们链接器什么时候会忽略LD\_PRELOAD环境变量，解释原因。

#####2.6.1 把myprog编译成一个普通用户下的程序在普通用户下运行

可见，它会使用LD_PRELOAD环境变量，重载sleep函数：

![tu-09](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-09.png)

#####2.6.2 把myprog编译成一个Set-UID root的程序在普通用户下运行

在这种情况下，忽略LD_PRELOAD环境变量，不重载sleep函数，使用系统自带的sleep函数：

![tu-10](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-10.png)

#####2.6.3 把myprog编译成一个Set-UID root的程序在root下运行

在这种情况下，使用LD_PRELOAD环境变量，使用重载的sleep函数：

![tu-11](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-11.png)

#####2.6.4在一个普通用户下把myprog编译成一个Set-UID 普通用户的程序在另一个普通用户下运行

在这种情况下，不会重载sleep函数：

![tu-12](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-12.png)

由以上四种情况可见：只有用户自己创建的程序自己去运行，才会使用LD\_PRELOAD环境变量，重载sleep函数，否则的话忽略LD\_PRELOAD环境变量，不会重载sleep函数。

####2.7 消除和清理特权

为了更加安全，Set-UID程序通常会调用setuid()系统调用函数永久的清除它们的root权限。然而有些时候，这样做是远远不够的。在root用户下，在/tmp目录新建一个空文件zzz。在root用户下将下面代码命名为test.c，放在/tmp目录下，编译这个程序，给这个程序设置root权限。在一个普通的用户下，运行这个程序。描述你所观察到的情况，/tmp/zzz这个文件会被修改吗？解释你的观察。

代码：

```
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
void main()
{
  int fd;
  //Assume that /tmp/zzz is an important system file,
  //and it is owned by root with permission 0644
  fd = open("/tmp/zzz", O_RDWR | O_APPEND);
  // Simulate the tasks conducted by the program
  sleep(1);
  // After the task, the root privileges are no longer needed,
   //it’s time to relinquish the root privileges permanently.
  setuid(getuid()); // getuid() returns the real uid
  if (fork())
  { // In the parent process
   close (fd);
    exit(0);
  }
  else
  { // in the child process
  //Now, assume that the child process is compromised, malicious
  //attackers have injected the following statements
  //into this process
   write (fd, "shiyanlou!", 10);
  close (fd);
  }
}
```

结果如图：

![tu-13](http://anything-about-doc.qiniudn.com/xxaq_set_uid/setuid-01-13.png)

如图所示文件被修改了，原因在于设置uid前，zzz文件就已经被打开了。只要将语句setuid(getuid())移至调用open函数之前，就能避免这个问题。

###三、练习

在实验楼环境安步骤进行实验，并截图


### License

本课程所涉及的实验来自[Syracuse SEED labs](http://www.cis.syr.edu/~wedu/seed/)，并在此基础上为适配[实验楼](http://www.shiyanlou.com)网站环境进行修改，修改后的实验文档仍然遵循GNU Free Documentation License。

本课程文档github链接：[https://github.com/shiyanlou/seedlab](https://github.com/shiyanlou/seedlab)

附[Syracuse SEED labs](http://www.cis.syr.edu/~wedu/seed/)版权声明：

> Copyright © 2014 Wenliang Du, Syracuse University.
The development of this document is/was funded by the following grants from the US National Science Foundation:
No. 1303306 and 1318814. Permission is granted to copy, distribute and/or modify this document
under the terms of the GNU Free Documentation License, Version 1.2 or any later version published by the
Free Software Foundation. A copy of the license can be found at http://www.gnu.org/licenses/fdl.html.


