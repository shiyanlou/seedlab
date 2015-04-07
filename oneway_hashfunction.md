##单向哈希函数与MAC
----

在线实验环境：[单向哈希函数与MAC](http://www.shiyanlou.com/courses/242)

###一、实验描述

本实验的学习目的是让学生熟悉单向哈希函数以及消息认证码（MAC），在完成本课实验后，学生应当对密码学概念有更深入的理解并且能通过使用工具或者编程为一段指定的信息生成哈希值以及消息认证码。

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

####实验1:生成消息摘要和MAC
在本实验中，我们将认识各种单向哈希算法。你可以使用openssl dgst 命令为一个文件生成哈希值。输入man openssl和man dgst查看手册。

	$ openssl dgst dgsttype filename

使用指定的单向哈希算法替换dgsttype部分，比如 -md5, -sha1, -sha256 等等，至少使用3种算法，描述你观测到的结果。

####实验2:Keyed Hash和HMAC

本节实验中，我们要为文件生成keyed hash（比如MAC）。使用-hmac选项，选项后所带参数就是key。

	$ openssl dgst -md5 -hmac "abcdefg" filename

请分别使用 HMAC-MD5, HMAC-SHA256 和 HMAC-SHA1 生成keyed hash，并配以不同长度的key。我们是否总是需要在HMAC中使用一个固定长度的key？如果是这样，key的长度是多少？如果不是，为什么？

####实验3：单向哈希函数的随机性

为了理解单向函数的特性，我们需要做一些关于 MD5 和 SHA256的练习
1. 创建一个任意长度的文本文件。
2. 使用指定的哈希算法为该文件生成哈希值H1。
3. 翻转该文件的任意一位，你可以使用ghex或者bless进行编辑。
4. 为修改后的文件生成哈希值H2。
5. 观察H1与H2是否相似，请描述你的观察。
6. 写一个程序计算H1与H2相同位的数量。

给出不能参考的样例，强迫症患者不用纠结这段，用自己的方式，不择手段完成实验练习就好：）
```python
# -*- coding:utf8 -*-
from os import urandom
import os, tempfile
import binascii


#翻转字节的任意一位
def flipBit(x, kth):
    x = ord(x)
    if (x & 1 << kth ) != 0:
        return chr(x & ~(1 << kth))
    else:
        return chr(x | ( 1 << kth))

#十六进制字符串转二进制字符串
def byte_to_binary(n):
    return ''.join(str((n & (1 << i)) and 1) for i in reversed(range(8)))

def hex_to_binary(h):
    return ''.join(byte_to_binary(ord(b)) for b in binascii.unhexlify(h))


origin_text = "hello, shiyanlou"    #源字符串
changed_text = list(origin_text)
changed_text[1] = flipBit(changed_text[1],4) 
changed_text = ''.join(changed_text) #修改后的字符串

with tempfile.NamedTemporaryFile() as fa:
    with tempfile.NamedTemporaryFile() as fb:
        #通过系统命令行命令获取hash值
        def get_hash(fp, text):
            fp.write(text)
            fp.seek(0)
            result = os.popen("openssl dgst -sha1 {}".format(fp.name)).read()[66:-1]
            print result
            return result

        origin_hash  = hex_to_binary(get_hash(fa, origin_text))
        changed_hash = hex_to_binary(get_hash(fb, changed_text))

        print origin_text + ":"
        print origin_hash
        print changed_text + ":"
        print changed_hash
        #计算不同的位数
        print "diff count:", reduce(lambda x, y: x+y, [origin_hash[i] == changed_hash[i] for i in range(len(origin_hash))])
```

####实验4：单向特性（One-Way）与避免冲突特性（Collision-Free）的对比

本节实验中，我们将探索哈希函数的两个特性 one-way 与 collision-free 。我们将使用暴力破解的方法看看攻破两种特性分别要花多长时间，这回我们不使用命令工具，而是调用加密库写C程序. 示例代码地址： http://www.openssl.org/docs/crypto/EVP_DigestInit.html

为了让实验短时间内可行，我们只使用哈希值的头24位。请自行设计实验完成下列问题：

1. 通过暴力破解，你需要尝试多少次才能够击破单向特性，你需要进行多次实验，计算平均值。
2. 同上，单向特性换成避免冲突特性。
3. 通过你的观察，哪种特性更容易通过暴力破解击破。
4. （Bonus！）你可以用数学方法解释你观察到的结果之间的区别吗？

注：为了编译你的代码或者写makefile，你可能需要以下信息。

>INC=/usr/local/ssl/include/
LIB=/usr/local/ssl/lib/
all:
	gcc -I\$(INC) -L$(LIB) -o enc yourcode.c -lcrypto -ldl


样例：
```
#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <openssl/evp.h>

int hash_table[16777216];
char buffer[50];
char hash_value[100];
long long count = 0;

int hex2int(char* hex)
{
  int i;
  int sum = 0;
  for (i = 0; i < strlen(hex); i+=2)
    sum *= 16;
    sum += 16 * (hex[i] - '0') + (hex[i+1] - '0');
  return sum;
}

void gen_string()
{
  srand(count++);
  sprintf(buffer, "%d", rand());
}

int main(int argc, char const *argv[])
{

  int one_way_flag = 0, co_free_flag = 0;
  clock_t start = clock();
  clock_t one_end, co_end;

  memset(hash_table, 0, sizeof(int) * 16777216);

  EVP_MD_CTX *mdctx;
  const EVP_MD *md;
  
  unsigned char md_value[EVP_MAX_MD_SIZE];
  int md_len, i;

  OpenSSL_add_all_digests();

  if(!argv[1]) {
    printf("Usage: mdtest hash_value\n");
    exit(1);
  }

  md = EVP_get_digestbyname(argv[1]);

  if(!md) {
    printf("Unknown message digest %s\n", argv[1]);
    exit(1);
  }

  char* search_value = argv[2];

  while(1){
    if (co_free_flag && one_way_flag) break;

    mdctx = EVP_MD_CTX_create();
    EVP_DigestInit_ex(mdctx, md, NULL);
    gen_string();
    EVP_DigestUpdate(mdctx, buffer, strlen(buffer));
    EVP_DigestFinal_ex(mdctx, md_value, &md_len);
    EVP_MD_CTX_destroy(mdctx);
    
    sprintf(hash_value, "%02x%02x%02x", md_value[0], md_value[1], md_value[2]);

    int idx = hex2int(hash_value);
    hash_table[idx]++;
    if (!co_free_flag && hash_table[idx] > 1)
    {
      co_free_flag = 1;
      co_end = clock();
      printf("Collision Found\ndelta time: %d\n", co_end - start);
    }

    if (0 == strcmp(search_value, hash_value)){
      one_way_flag = 1;
      time(&one_end);
      printf("Content Found. The content is %s\ndelta time: %d\n", buffer, one_end - start);
      break;
    }
  }
  /* Call this once before exit. */
  EVP_cleanup();
  exit(0);
}

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






