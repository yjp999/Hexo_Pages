---
title: AES算法详解
date: 2016-10-11 20:14:47
tags: [密码学,c]
categories: [编程]
---

## AES背景简介

> 高级加密标准，在密码学中又称Rijndael加密法，是美国联邦政府采用的一种区块加密标准。              —— [维基百科](https://zh.wikipedia.org/wiki/高级加密标准)

曾经广泛使用的DES久负盛名，因为它的56位密钥过短（再加上8位校验码，也称为64位密钥），已被AES逐渐取代。在计算机的升级换代后，其运算速度大幅度提高， 破解DES密钥所需时间也将越来越短，于是在2000年10月，NIST（National Institute of Standords and Technology）选择了新的密码——高级加密标准AES，用于替代DES。

无论是之前的DES还是现在广泛使用的AES，都属于对称密码技术。对称密码技术和公开密钥密码技术（如著名的RSA）相比，加密密钥和解密密钥是相同的，其最大的优势就是速度快，一般用于**大量数据**的加密和解密。

## AES算法描述
### 原理详述
AES算法是基于置换和代替的，置换是数据的重新排列，而代替是用一个单元数据替换另一个。AES使用了几种不同的技术来实现置换和替换，包括：

- **SubBytes** ：字节代换，属于非线性变换，独立地将状态的每个字节进行。代换表（S-盒）是可逆的。状态矩阵按照下面的方式被映射成为一个新的字节：
> 将该字节的高4位作为行值，低4位作为列值，得到S盒或逆S盒的对应元素作为输出。

例如输入字节0x12，取S盒的第0x01行第0x02列，得到0xC9。
{% asset_img aes_ByteSub.png ByteSub %}

----------
- **ShiftRows** ：行循环移位，在行循环移位变换中，状态阵列的后三行循环移位不同的偏移量。第0行不移动，第1行循环移位C1字节，第2行循环移位C2字节，第3行循环移位C3字节。偏移量C1、C2、C3与分组长度 $N_{b}$ 有关。如下表所示.

| $N_b$ |  C1  |  C2  |  C3  |
|:---:  | :---:|:---: |:---: |
| 4     | 1    | 2    | 3    |
| 6     | 1    | 2    | 3    |
| 8     | 1    | 3    | 4    |

行移位示意图：
{% asset_img aes_ShiftRows.png ShiftRows %}

----------
- **MixColumn** ：列混合运算，该运算将状态(State)的列看作是有限域$GF(2^8)$上的多项式$a(x)$，与多项式$c(x)$相乘（在模（$x^4+1$）下）。运算公式如下：

\begin{equation} 
	b(x)=c(x)*a(x)  \mod (x^4+1)
\end{equation}

$$ \begin{bmatrix}
b_0 \\\\
b_1 \\\\
b_2 \\\\
b_3
\end{bmatrix} = \begin{bmatrix}
02&03&01&01 \\\\
01&02&03&01 \\\\
01&01&02&03 \\\\
03&01&01&02 
\end{bmatrix} * \begin{bmatrix}
a_0\\\\
a_1\\\\
a_2\\\\
a_3
\end{bmatrix} $$

列混合运算示意图
{% asset_img aes_MixColumn.png MixColumn %}

----------
- **AddRoundKey** ：密钥加，它是将轮密钥简单地与状态进行逐比特异或。轮密钥由种子密钥通过密钥编排算法得到，轮密钥长度等于分组长度$N_b$。
密钥加运算示意图：
{% asset_img aes_AddRoundKey.png AddRoundKey %}

### AES的密钥调度
* 密钥bit的总数 = 分组长度 x （轮数Round + 1）
* 当分组长度是128bit且轮数为10时，轮密钥长度为$128 * (10+1) = 1408 bit$
* 将初始密钥扩展成扩展密钥
* 轮密钥从扩展密钥中取，第1轮轮密钥取扩展密钥的前$N_b$个字，第2轮轮密钥取接下来的$N_b$个字，以此类推。
### 密钥扩展
AES密钥扩展图如下：
{% asset_img aes_KeyExtent.png KeyExtent %}
* 函数$T$由三部分组成：字循环移位、字节代换和轮常量异或。
	* 字循环移位：将1个字中的4个字节循环左移1个字节，即将输入字$[b_0, b_1 , b_2 , b_3]$变换为$[b_1, b_2 , b_3 , b_0]$。
	* 字节代换：对字循环的结果使用S盒进行字节代换。
	* 轮常量异或： 将前两步的结果同轮常量$Rcon[j]$进行异或，其中$j$表示轮数。
* 轮常量是一个字，使用轮常量是为了防止不同轮中产生的轮密钥的对称性或相似性。

### AES小结
* AES的密钥长度和加密轮数列表如下：

|name     |密钥长度(32bit)$(N_k)$|分组长度$N_b$ |加密轮数$(N_r)$ |
|:---:    | :---:              |:----:       | :--:         | 
| AES-128 | 4                  | 4           | 10           |
| AES-192 | 6 				   | 4			 | 12		    |
| AES-256 | 8 				   | 4			 | 14		    |

显然，AES是一个迭代的对称密钥分组的密码，它可以使用128,192和256位密钥，并且用128位（16字节）分组加密和解密数据。
* AES的加密/解密流程图
{% asset_img aes_flow.png The flow of encryption and decryption in AES %}

## AES的应用
研一第一学期我们院系除了别的课程，还开了两门课，现代密码学和网络信息安全，这两门课程我也都选择了，因为均有涉及到密码基础学，并且网络信息安全这门课的老师在上周留了一个project，写一个程序要求利用AES算法实现对文本文件的内容加解密，具体要求如下：
* 从命令行接收3个参数
* 参数1=enc表示加密，参数1=dec表示解密
* 参数2为待加密、解密的文件名
* 参数3为密码

为了快速实现这一功能，我直接调用了openssl库里的现有AES算法，整个程序的代码如下：

``` c++
// Linux: gcc -o encfile encfile.cpp -lcrypto

#include <memory.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/mman.h>

#define N 4096

#include "openssl/aes.h"

#pragma comment(lib,"libeay32.lib")

unsigned char buf[16];
unsigned char buf2[16];
unsigned char aes_keybuf[32];
AES_KEY aeskey;


void encode(char inString[], int inLen, char passwd[], int pwdLen)
{
	int i,j, len, nLoop, nRes;
	char enString[N];

	// 准备32字节(256位)的AES密码字节
	memset(aes_keybuf,0x90,32);
	if(pwdLen<32){ len=pwdLen; } else { len=32;}
	for(i=0;i<len;i++) aes_keybuf[i]=passwd[i];
	// 输入字节串分组成16字节的块	
	nLoop=inLen/16; nRes = inLen%16;
	// 加密输入的字节串
	AES_set_encrypt_key(aes_keybuf,256,&aeskey);
	for(i=0;i<nLoop;i++){
		memset(buf,0,16);
		for(j=0;j<16;j++) buf[j]=inString[i*16+j];
		AES_encrypt(buf,buf2,&aeskey);
		for(j=0;j<16;j++) enString[i*16+j]=buf2[j];
	}
	if(nRes>0){
		memset(buf,0,16);
		for(j=0;j<nRes;j++) buf[j]=inString[i*16+j];
		AES_encrypt(buf,buf2,&aeskey);
		for(j=0;j<16;j++) enString[i*16+j]=buf2[j];
		//puts("encrypt");
	}
	enString[i*16+j]=0;
	printf("The encrypted string is:\n  %s ", enString);

	FILE *fp;
	fp = fopen("enc_string.txt","w");
	if(fp==NULL){
		printf("Failure to open the file.\n");
	}else{
		fwrite(enString, 1, sizeof(enString), fp);
	}
	fclose(fp);
	// return enString;
}

void decode(char *enStr, int inLen, char passwd[], int pwdLen)
{
	int i,j, len, nLoop, nRes;
	char deString[N];

	// 准备32字节(256位)的AES密码字节
	memset(aes_keybuf,0x90,32);
	if(pwdLen<32){ len=pwdLen; } else { len=32;}
	for(i=0;i<len;i++) aes_keybuf[i]=passwd[i];
	// 输入字节串分组成16字节的块	
	nLoop=inLen/16; nRes = inLen%16;

	// 密文串的解密	
	AES_set_decrypt_key(aes_keybuf,256,&aeskey);
	for(i=0;i<nLoop;i++){
		memset(buf,0,16);
		for(j=0;j<16;j++) buf[j]=enStr[i*16+j];
		AES_decrypt(buf,buf2,&aeskey);
		for(j=0;j<16;j++) deString[i*16+j]=buf2[j];
	}
	if(nRes>0){
		memset(buf,0,16);
		for(j=0;j<16;j++) buf[j]=enStr[i*16+j];
		AES_decrypt(buf,buf2,&aeskey);
		for(j=0;j<16;j++) deString[i*16+j]=buf2[j];
		//puts("decrypt");
	}
	deString[i*16+nRes]=0;
	printf("The decrypted string is:\n  %s ", deString);
	
}

int main(int argc, char* argv[])
{

	char *inString;
	if(argc != 4){
		printf("usage: %s <purpose> <filename> <password>\n", argv[0]);
	}

	char *purpose = argv[1];
	char *filename = argv[2];
	char *passwd = argv[3];

	struct stat sb;
	int fd;
	fd = open(filename, O_RDONLY);
	fstat(fd, &sb);

	inString = (char *)mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
	close(fd);

	if(strcmp(purpose, "enc") == 0){
		encode(inString, strlen(inString), passwd, strlen(passwd));	
	}else if(strcmp(purpose, "dec") == 0){
		decode(inString, strlen(inString), passwd, strlen(passwd));
	}else{
		fprintf(stderr, "the first argument must be 'enc' or 'dec' %s\n", argv[0]);
	}

	return 0;
}
```
代码的组织逻辑非常简单:
* **main** 函数：接收命令行参数，并读取指定的文件内容。
* **encode **函数：对字符串利用openssl库里的aes算法加密（使用命令行参数3提供的密码）。
* **decode** 函数：对文本文件里的加密串进行解密，由密文转换成明文。

需要注意的是，在Linux下面利用openssl库进行开发之前，必须保证已安装openssl，否则编译报错。
安装openssl的命令是：
``` bash
sudo apt-get install openssl
sudo apt-get install libssl-dev
```
编译源码时，需要在后面添加**-lcrypto**，因为在链接时需要用到linux下的加密库。假设源代码文件名为encfile.cpp，编译命令：
``` bash
gcc -o encfile encfile.cpp -lcrypto
```

## 总结

在写这篇博客的过程中，花费了我很多精力和时间。其实在大三下阶段我就上过关于密码学的课程，但是由于当时逃课成瘾，课上几乎没怎么听，只是简单地应付了一下考试，所以关于密码学的相关知识基础非常薄弱，于是趁着现在事情不算多，又在现阶段重新选了密码学的课程，就把最近在密码学里面刚学到的的AES算法进行一个总结并实践之。
