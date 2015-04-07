##密钥加解密实验
----

在线实验环境：[密钥加解密实验](http://www.shiyanlou.com/courses/241)

###一、实验描述

本实验的学习目的是让学生熟悉加密的概念，熟悉和了解加密算法(cipher)、加密模式(encryption mode)、 填充(padding)、以及初始向量（IV）的定义与作用。此外，学生还可以通过使用工具和编写程序加密/解密信息。

###二、实验环境

本实验中，我们将使用openssl命令行工具及其库。实验环境中已自带命令行工具，需安装openssl开发库。
```python
$ sudo apt-get update
$ sudo apt-get install libssl-dev
```
编辑器使用bless十六进制编辑器,需预先安装。
```python
$ sudo apt-get install bless
```
系统用户名:	shiyanlou，密码shiyanlou

###三、实验内容

####实验1. 使用不同的加密算法与加密模式进行加密
你可以使用openssl enc 命令来加密／解密一个文件。输入man openssl和man enc命令以查看手册

```
$ openssl enc ciphertype -e -in plain.txt -out cipher.bin \
-K 00112233445566778899aabbccddeeff \
-iv 0102030405060708
```

请将上述代码中的 ciphertype替换成指定的加密类型，比如-aes-128-cbc, -aes-128-cfb, -bf-cbc 等,在本实验中你至少需要使用三种不同的密码(cypher)与加密模式(encryption mode)可以通过输入man enc 命令了解到各选项的含义以及openssl所支持的所有加密类型。

密码(cipher):指 bf, cast, des, aes, rs2 等
加密模式(encryption mode):指 cbc, ecb, cfb, ofb 等

以下列出部分主要选项释义：


| 参数        | 含义      		| 含义(中文) |
| ---------- |---------------   |-----    | 
|-in 		 |<file> input file |输入文件	   |
|-out        |<file> output file|输出文件    |
|-e          |encrypt 			|加密       |
|-d          |decrypt 			|解密       |
|-K/-iv      |key/iv in hex is the next argument |十六进制密钥与初始向量|
|-[pP] |print the iv/key (then exit if -P) |打印初始向量与密钥|

####实验2. 加密模式——ECB 与CBC的区别。
![](http://www.cis.syr.edu/~wedu/seed/Labs_12.04/Crypto/Crypto_Encryption/files/pic_original.bmp)

[图片地址](http://www.cis.syr.edu/~wedu/seed/Labs_12.04/Crypto/Crypto_Encryption/files/pic_original.bmp)

pic_original.bmp是一张简单的图片，我们将对它进行加密，这样没有密钥的人就无法看到这张图是什么了。请分别使用ECB和CBC模式。
1. 加密后的图片bmp文件头是损坏的，我们需要用bless编辑该图片文件。（注：bmp头长度为54字节）

2. 修改完后，打开图片，描述你的观察结果。

修改加密后文件的文件头
![](http://i.imgur.com/ssMTTDx.png)

观察结果(左:cbc 中:ecb 右:原图)
![](http://i.imgur.com/Fc2NBrY.png)

####实验3:  加密模式——损坏的密文

为了理解各种不同加密模式的特性，我们需要做以下练习：
1. 创建一个长度至少64字节的文本文件。
2. 使用AES-128对其进行加密
3. 不幸的是，密文的第30位损坏了。请用编辑器模拟损坏，编辑该位，将其反转。
4. 使用相同的key与iv对密文进行解密。


1.分别用ECB，CBC，CFB，OFB四种模式解密损坏的文件，观察有多少数据是正确恢复的，造成这种结果的原因是什么？

![](http://i.imgur.com/OGwpH6k.png)
(注：test.txt为原文件，test_(模式)_d.txt为对应解密文件)

####实验4: 填充

对于分组密码来说，当明文大小不是分组大小的倍数的时候，就需要使用填充了。请做以下练习：

1.openssl手册（man openssl）中提到openssl使用pkcs5标准来做填充。请在查询pkcs5标准给出的填充方法后设计实验验证这一点。

下面给出不能参考的样例

```python
# -*- coding:utf8 -*-
from os import urandom
import os, tempfile

#明文
plaintext = "hello shiyanlou!"

#生成key与iv的伪随机数
key = urandom(16)
iv = urandom(16)

#未手动填充前加密
with tempfile.NamedTemporaryFile() as fa:
    with tempfile.NamedTemporaryFile() as fb:
        cmd = "openssl enc -aes-128-ecb -in {0} -out {1} -K {2} -iv {3}".format(fa.name, fb.name, str(key.encode("hex")), str(iv.encode("hex")))
        os.system(cmd)
        openssl = fb.read().encode("hex")

print "key:", key.encode("hex")
print "iv: ", iv.encode("hex")
print "plaintext: ", plaintext
print "填充前(hex):", plaintext.encode("hex")

#手动填充
length = 16 - (len(plaintext) % 16)
plaintext +=chr(length)*length

with tempfile.NamedTemporaryFile() as fa:
    with tempfile.NamedTemporaryFile() as fb:
        cmd = "openssl enc -aes-128-ecb -in {0} -out {1} -K {2} -iv {3}".format(fa.name, fb.name, str(key.encode("hex")), str(iv.encode("hex")))
        os.system(cmd)
        openssl2 = fb.read().encode("hex")

print "填充后(hex):", plaintext.encode("hex")

print "openssl(未手动填充):", openssl
print "openssl(手动填充后):", openssl2
```

2.使用ECB，CBC，CFB，OFB四种模式加密同一个文件，说明哪种模式会使用填充，哪种模式不会，为什么有些模式不需要填充。


####实验5:使用Openssl加密库进行编程

上节实验中我们已经学习了如何使用openssl来加密和解密信息。在本实验中，我们将学习如何使用openssl的加密库来加密／解密信息，openssl提供一个叫作EVP的高级接口，已封装底层函数，尽管openssl也为每一个单独的加密算法提供接口，但使用evp会更方便一些。

以下地址给出样例代码，先自行熟悉EVP接口，再继续接下来的实验。
https://www.openssl.org/docs/crypto/evp.html

实验已给出明文和密文，并告知加密方法为aes-128-cbc，IV全由0组成，以及key的长度小于16个字母，该单词可以从一般的英文词典中得到。由于该单词小于16个字母（128bits）所以在其后追加了空格字符（对应0x20）以达到128bit的长度。给你的任务就是写一个程序找到key，你可以从网上下载一个英文词典，也可以使用我们提供的。
[词典地址](http://www.cis.syr.edu/~wedu/seed/Labs_12.04/Crypto/Crypto_Encryption/files/words.txt)

>明文 (21个字符): This is a top secret.
>密文 (十六进制形式): 8d20 e505 6a8d 24d0 462c e74e 4904 c1b5 13e1 0d1d f4a2 ef2a d454 0fae 1ca0 aaf9

Note 1:	 如果你打算将明文存储在文件中读取，一些编辑器可能会擅自加奇怪的东西进去，比如windows的记事本，使用实验室环境的编辑器不会有这样的顾虑。

Note 2: 为了编译你的代码或者写makefile，你可能需要以下信息。

>INC=/usr/local/ssl/include/
LIB=/usr/local/ssl/lib/
all:
	gcc -I\$(INC) -L$(LIB) -o enc yourcode.c -lcrypto -ldl

下面给出不能参考的样例代码
```
#include <openssl/conf.h>
#include <openssl/evp.h>
#include <openssl/err.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

#define True 1
#define False 0

void handleErrors(void)
{
  ERR_print_errors_fp(stderr);
  abort();
}

int encrypt(unsigned char *plaintext, int plaintext_len, unsigned char *key,
  unsigned char *iv, unsigned char *ciphertext)
{
  EVP_CIPHER_CTX *ctx;

  int len;

  int ciphertext_len;

  if(!(ctx = EVP_CIPHER_CTX_new())) handleErrors();

  if(1 != EVP_EncryptInit_ex(ctx, EVP_aes_128_cbc(), NULL, key, iv))
    handleErrors();

  if(1 != EVP_EncryptUpdate(ctx, ciphertext, &len, plaintext, plaintext_len))
    handleErrors();
  ciphertext_len = len;

  if(1 != EVP_EncryptFinal_ex(ctx, ciphertext + len, &len)) handleErrors();
  ciphertext_len += len;

  EVP_CIPHER_CTX_free(ctx);

  return ciphertext_len;
}

int append(char* buffer){
    int length = (int)strlen(buffer);
    if (length > 16)    
        return False;
    memset(buffer+strlen(buffer),' ', 16-length);
    buffer[16] = '\0';
    return True;

}

int main(int argc, char const *argv[])
{
    char buffer[50];

    int i = 0;
    char iv[17];
    memset(iv, 0, 17);

    unsigned char *plaintext = "This is a top secret.";
    unsigned char ciphertext[100];
    unsigned char *cryptotext="8d20e5056a8d24d0462ce74e4904c1b513e10d1df4a2ef2ad4540fae1ca0aaf9";
                    
    ERR_load_crypto_strings();
    OpenSSL_add_all_algorithms();
    OPENSSL_config(NULL);

    int ciphertext_len;

    FILE* fp = fopen("words.txt", "r");
    while (fscanf(fp, "%s\n", buffer) != EOF){
        if (!append(buffer))
            continue;
        
        ciphertext_len = encrypt(plaintext, strlen(plaintext), buffer, iv, ciphertext);

        unsigned char cryptohex[50];
        for (i = 0; i < ciphertext_len; i++)
        {
            sprintf(cryptohex+i*2,"%02x", ciphertext[i]);
        }   
        cryptohex[ciphertext_len*2] = '\0';

        if (0 == strcmp(cryptohex, cryptotext)){
            printf("The key is: %s\n", buffer);
            break;
        }   
    }
    EVP_cleanup();
    ERR_free_strings();
    return 0;
}

``` 

####实验6：生成伪随机数
生成随机数在软件开发中很常见，在多数情况下，加密密钥并不由用户提供，而是由软件本身生成。随机数的随机性很重要，否则，攻击者就可能预测密钥，产生严重的后果。大多数开发者知道如何去生成随机数（比如。蒙特卡罗方法）。所以他们以相似的方法生成随机数。但这些“随机数”事实上并不够随机，仍然可能被攻击破解，历史上Netscape 和 Kerberos 就犯过这样的错误。

在本实验中，学生将学习使用标准方法生成能够达到安全目的的伪随机数。

#####6.A：内核熵（entropy）
计算机本身并不适合生成随机数，所以大多数系统通过物理资源获得随机性。比如linux通过以下函数获得随机性。
```
void add_keyboard_randomness(unsigned char scancode);
void add_mouse_randomness(__u32 mouse_data);
void add_interrupt_randomness(int irq);
void add_blkdev_randomness(int major);
```

前两个很好理解，第一个利用键盘按键时序和所按键的对应码，第二个利用鼠标的移动和中断时序。第三个利用所有的中断时序收集随机性，当然，并不是所有的中断都有好的随机性，比如时间中断就是可以预测的，而硬盘中断是一个好选择，第四个函数计算设备块请求的完成时间。

我们使用熵来衡量随机性，在这里熵只意味着计算机当前拥有多少位随机比特。以下命令可以得到系统当前拥有熵的数量：

	$ cat /proc/sys/kernel/random/entropy_avail

Please move and click your mouses, type somethings, and run the program again. Please describe your
observation in your report.
请移动或者点击鼠标，敲敲键盘，拿电脑闷泡面或者对电脑做点别的什么，之后再一次运行上面的命令，描述你的观察。

#####6.B： 从 /dev/random 获得伪随机数。
linux 将从物理世界得到的随机数据存于一个随机池中，再由两个设备把随机数据转化成伪随机数。这两个设备有着不同的行为，我们先来学习/dev/random。

你可以使用以下命令从/dev/random得到16字节的伪随机数，我们把数据pipe到hexdump中查看内容。

	$ head -c 16 /dev/random | hexdump

请多次运行该命令，你会发现命令发生阻塞，因为每次取出伪随机数都会导致随机池的熵减少，当熵用完的时候，设备就会被阻塞直到获得足够的熵。请做点什么解决阻塞，让命令打出随机数来。

#####6.C 从/dev/urandom 获得伪随机数。

此设备不会被阻塞，即使随机池的熵已经相当低了。
你可以使用以下命令从/dev/urandom获得1600字节的伪随机数，运行几次检测该设备是否会阻塞。

	$ head -c  1600 /dev/urandom | hexdump

/dev/random 与 /dev/urandom 都是从池中取出随机数据来生成伪随机数的。当熵不够用的时候，/dev/random会阻塞，而/dev/urandom则会持续生成新的数。请把池中的数据看作种子（seed），我们都知道，一颗种子能够生成任意数量的伪随机数。理论上来说, /dev/random设备更加安全，不过在日常实践中，二者并没有太大区别，因为种子是随机且不可预测的,在取得新的随机数据时该设备就会补种，反而是/dev/random设备的阻塞可能导致拒绝服务攻击。

所以推荐使用/dev/urandom 来获得随机数，为了在程序中使用它，你只需要直接从设备文件中读取。下面的代码片段演示设备的使用方法。

```
#define LEN 16 // 128 bits
unsigned char *key = (unsigned char *) malloc(sizeof(unsigned char)*LEN);
FILE* random = fopen("/dev/urandom", "r");
fread(key, sizeof(unsigned char)*LEN, 1, random);
fclose(random);
```


###四、作业

####按要求完成实验内容并回答每节实验给出的问题。

### License

本课程所涉及的实验来自[Syracuse SEED labs](http://www.cis.syr.edu/~wedu/seed/)，并在此基础上为适配[实验楼](http://www.shiyanlou.com)网站环境进行修改，修改后的实验文档仍然遵循GNU Free Documentation License。

本课程文档github链接：[https://github.com/shiyanlou/seedlab](https://github.com/shiyanlou/seedlab)

附[Syracuse SEED labs](http://www.cis.syr.edu/~wedu/seed/)版权声明：

> Copyright © 2014 Wenliang Du, Syracuse University.
The development of this document is/was funded by the following grants from the US National Science Foundation:
No. 1303306 and 1318814. Permission is granted to copy, distribute and/or modify this document
under the terms of the GNU Free Documentation License, Version 1.2 or any later version published by the
Free Software Foundation. A copy of the license can be found at http://www.gnu.org/licenses/fdl.html.








