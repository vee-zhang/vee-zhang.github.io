---
title: "C语言简明教程"
subtitle: ""
date: 2022-10-17T15:49:18+08:00
lastmod: 2022-10-17T15:49:18+08:00
draft: false
authors: []
description: ""

tags: []
categories: []
series: []

---

<!--more-->

### 基本语法

```c
// 引入头文件,这个是所有C语言编译器的标准部分，用来提供输入和输出的支持。
// include语句是C的预处理器指令，#符号标注的不是c的语句，而是表示在编译之前，由预处理器来处理的语句。
#include<stdio.h>

//主函数
//返回值是返回给操作系统的
int main(void){
    int number;
    number = 2014;
    printf("Hello ! I am dotcpp.com\n");
    print("This tear is %d\n",number);
    return 0;
}
```

- 变量名区分大小写，允许大小写字母、数字、下划线，第一个字符不能是数字。

### 数据类型

| 类型         | 关键字          | 占用字节数 |
| ------------ | --------------- | ---------- |
| 字符         | char            | 1          |
| 无符号字符   | unsignedchar    | 1          |
| 整数         | int             | 4          |
| 无符号整数   | unsigned int    | 4          |
| 短整数       | short           | 2          |
| 无符号短整数 | unsignved short | 2          |
| 长整数       | long            | 4          |
| 无符号长整数 | unsigned long   | 4          |
| 单精度浮点   | float           | 4          |
| 双精度浮点   | double          | 8          |
| 长双精度浮点 | long double     | 10         |

#### 类型转换

```
(type_name) expression
```

eg:

```
(float) a;  //将变量 a 转换为 float 类型
(int)(x+y);  //把表达式 x+y 的结果转换为 int 整型
(float) 100;  //将数值 100（默认为int类型）转换为 float 类型
```

### 数组

```
类型说明符 数组名 [常量表达式];
```

```c
int a[1000]={1,2,3,4,5};
float b[10];
int a[3][4]={ {1,2,3}, {10,20,30}};
```

### 输入与输出

- `printf`
- `putchar`输出单字符。
- `getchar`接收键盘输入的一个字符到变量。

#### 格式化打印

```c
printf("格式控制字符串",输出表项);
```

返回可打印的字符的数量，如果输出错误，将返回-1。

| 占位符 | 解释                                               |
| ------ | -------------------------------------------------- |
| d,i    | 以十进制形式输出有符号整数                         |
| O      | 以八进制形式输出无符号整数（不含前缀0）            |
| x      | 以十六进制形式输出无符号整数（不含前缀0x）         |
| U      | 以十进制形式输出无符号整数                         |
| f      | 以小鼠形式输出单、双精度实数                       |
| e      | 以指数形式输出单、双精度实数                       |
| g      | 以%f或%e中较短输出宽度的一种格式输出单、双精度实数 |
| C      | 输出单字符                                         |
| S      | 输出字符串                                         |

#### puts

```c
#include <stdio.h>
int main(void)
{
    char str[100] = "www.dotcpp.com";
    printf("%s\n", str);  
    puts(str);  
    return 0;
}
```                                       |


#### 接收单字符输入

```c
char c;//定义个字符变量c
c=getchar();//将读取到的字符赋值给字符变量c
```

#### scanf

```c
scanf("格式控制字符串",输入项地址列表);
```

格式控制字符串的作用于printf函数相同，但不能显示非格式字符串，也就是不能显示提示字符串；地址表项中的**地址**要给出变量的地址，由`&`后跟变量名组成。

#### get

```c
# include <stdio.h>
int main(void)
{
    char str[100] = "\0";  
    printf("请输入字符串：\n");
    gets(str);
    printf("刚才输入的字符串是：\n");
    printf("%s\n", str);
    return 0;
}
```

get方式不会对输入进行检查和验证，所以输入内容过长可能会造成OOM，所以并不安全。

### sizeof运算符

这玩意不是函数，而是c语言自带的32个关键字之一，叫做**长度（求字节）运算符**，用来计算**类型**或者**变量**占用内存的字节数。

由于不是函数，所以`()`并不是必须的，比如后面跟的是变量就可以省略`()`，而如果跟的是类型，则必须`()`。

```c
#include<stdio.h>
int main()
{
    int n=0;
    int intsize = sizeof(int);
    printf("int sizeof is %d bytes\n",intsize);
    return 0;
}
```

### 三目运算符

```c
2>1?10:20
```

### 逻辑语句

#### if语句

```c
if(表达式) /*如果表达式成立，执行语句1否则继续判断表达式2*/ 
{ 
    //语句1 
} 
else if(表达式2) /*如果表达式成立，执行语句2否则继续判断表达式3*/ 
{ 
    //语句2 
}
else /*如果以上表达式都不成立 则执行语句4*/ 
{ 
    //语句4 
}
```

#### switch语句

```c
switch(表达式) /*首先计算表达式的值*/ 
{ 
    case 常量表达式1:语句1; 
    break;
    case 常量表达式2:语句2; 
    break;
    case 常量表达式3:语句3; 
    break;
    // …… 
    case 常量表达式n:语句n; 
    default:语句n+1;
}
```

#### while循环

```c
while(表达式) 
{ 
    循环体语句 
}
```

#### dowhile循环

```c
do 
{ 
    循环体语句 
}while(表达式);
```

#### for循环

```c
for(初始化表达式;判断表达式;更新表达式)
{
    循环体语句 
    continue;
}
```

### 函数

```c
返回值类型 函数名(形参表说明) /*函数首部*/
{
    说明语句 /*函数体*/
    执行语句
}
```

注意：可以不指明返回值类型，这样系统**默认返回整型**。

### 变量储存

从时间角度分为**静态储存**和**动态储存**两种：

- 静态储存，变量储存在内存的静态储存区，在编译时就分配了储存空间，在整个程序的运行期间，该变量占有固定的储存单元，在程序结束后，这部分空间才会释放，变量的值在整个程序中始终存在。
- 动态储存，变量在储存在内存的动态储存区，在程序的运行过程中，只有当变量所以在的函数被调用时，编译系统才临时为该变量分配一段内存单元，函数调用结束，该变量空间释放，变量的值旨在函数调用期内存在。

变量的作用范围又分为**局部变量**和**全局变量**：

#### 全局变量

外部变量就是全局变量，是在函数外部定义的。他的作用于从变量定义处开始，到程序文件的末尾。**如果在定义点之前的函数想饮用该外部变量，则应该在引用处用关键字`extern`对该变量进行声明**，就可以合法的使用该变量了。

#### 自动变量

函数中的局部变量、函数中的形参，在调用时函数时系统会给他们分配储存空间，在函数调用后会自动释放这些空间，这类变量称为**自动变量**。

```c
int fun(int a)
{
    auto int b,c=3;/*定义b,c为自动变量*/
}
```

#### 静态变量

```c
#include<stdio.h>
static a=5;
int fun()
{
    a=a*2;
    printf("a=%d\n",a);
    return 0;
}
int main()
{
    int i=0;
    for(i=0;i<10;i++)
    {
        fun();
    }
    return 0;
}
```

#### 寄存器变量

为提高效率，c语言允许将局部变量的值存放到cpu的寄存器中，叫做**寄存器变量**，用关键字**register**声明。

注意：

- 只有局部自动变量和形参可以作为寄存器变量。
- 一个计算机系统中的寄存器数量有限，不能定义任意多个寄存器变量。
- 不能使用地址运算符`&`求寄存器变量的地址。

```c
register int a=0;//将变量a存储在寄存器上
```

### 指针

用`&`符号表示指针或引用地址，用`*`符号获取指针对应的值。

```c
#include<stdio.h>
int main()
{
        int num=2014;
        int *p=&num;
        printf("num Address = 0x%x,num=%d\n",&num,num);
        printf("p = 0x%x,*p=%d\n",p,*p);
        printf("%d\n",*&num);
        return 0;
}
```

### 字符串处理

字符串初始化：

```
char *str = "www.dotcpp.com" ;
```

可以用字符串常量对指针进行初始化。

| 函数   | 用途                   |
| ------ | ---------------------- |
| strcpy | 实现两个字符串的拷贝   |
| strcat | 实现两个字符串的拼接   |
| strcmp | 对比两个字符串是否相等 |
| strlen | 计算字符串长度         |

### 结构体

声明：

```c
struct 结构类型名 
{ 
    数据类型 成员名 1; 
    数据类型 成员名 2; 
    ...... 
    数据类型 成员名 n; 
};
```

初始化：

```c
struct 结构类型名 结构变量 = { 初始化数据 1, ...... 初始化数据 n };
```

示例：

```c
#include<stdio.h>
#include<string.h> 
struct _INFO
{
    int num;
    char str[256];
};
int main() 
{
    struct _INFO A;
    A.num = 2014;
    strcpy(A.str,"Welcome to dotcpp.com");
    printf("This year is %d %s\n",A.num,A.str);
    return 0;
}
```

#### 结构体指针

结构体变量用`.`来访问成员，而结构体指针用`->`来访问。

示例：

```c
#include<stdio.h>
struct address
{
    char name[30]; /*姓名，字符数组作为结构体中的成员*/
    char street[40]; /*街道*/
    unsigned long tel; /*电话，无符号长整型作为结构体中的成员*/
    unsigned long zip; /*邮政编码*/
};
int main()
{
    struct address A[3]={"Zhang","Road NO.1",111111,4444},
    {"Wang"," Road NO.2",222222,5555},
    {"Li"," Road NO.3",333333,6666}};
    struct address *p;
    p=&A[0];
    printf("%s %s %u %u\n",p->name,p->street,p->tel,p->zip);
    return 0;    
}
```

### 共用体

在C语言中，允许几种不同类型的变量存放到同一段内存单元中，也就是使用覆盖技术，几个变量互相覆盖。这种几个不同的变量共同占用一段内存的结构，被称为共用体类型结构，简称共用体。一般定义形式为：

```c
union 共用体名 
{ 
    数据类型 成员名 1; 
    数据类型 成员名 2; 
    ...... 
    数据类型 成员名 n; 
}变量名表列;
```

- []暂时不知道有啥用，待学吧～

### typedef

可以用来定义新的类型,或者说给类型起个别名：

```
typedef 已定义的类型 新的类型;
```

例如：

```c
typedef int INTEGER; /*指定用 INTEGER 代表 int 类型*/
typedef float REAL; /*指定用 REAL 代表 float 类型*/
```

### 枚举

```
enum　枚举名　{枚举元素1,枚举元素2,……};
```

如：

```c
#include<stdio.h>
int main()
{
    enum Week{MON, TUE, WED, THU, FRI, SAT, SUN};//定义一个enum week类型，七个值（0.1.2...6）
    enum Week A=WED; //第三个值赋给A
    printf("%d\n",A);  
    return 0;
}
```

枚举类型的特点默认是**从0开始递增**，若想更改，可以将其中某个类型赋值，后面的值将在此基础之上递增，如代码：

```c
#include<stdio.h>
int main()
{
    enum Week{MON, TUE=5, WED, THU, FRI, SAT, SUN};
    enum Week A=WED;
    printf("%d\n",A);
    return 0;
}
```

### 预处理

#### 宏

分为**有参宏**和**无参宏**，

##### define

`define`关键字用来自定义宏：

无参宏：

```
#define 标识符 字符串or表达式;
```

有参宏：

```
#define 宏名(形参表) 字符串or表达式;
```

如：

```c
#define M (y*y+3*y);
```

这个宏定义的意思是：使用M替代后面的表达式。

示例：

```c
#include <stdio.h>
#define MAX(a,b) (a>b)?a:b
/*带参数的宏定义*/
main()
{
    int x,y,max;
    printf("input two numbers: ");
    scanf("%d %d",&x,&y);
    max=MAX(x,y);
    printf("max=%d\n",max);
    /*宏调用*/
}</stdio.h>
```

#### include

用来把指定的文件与当前的源程序文件连成一个源文件。

```c
#include "stdio.h"
#include <stdio.h>
```

- 尖括号表示在包含文件目录中去查找。
- 双引号表示在当前的源文件目录中查找，找不到则去包含文件目录中去找。

#### 条件编译

```c
#ifdef 标识符
程序段 1
#else
程序段 2
#endif
```

如果标识符已被`#difine`命令定义过，则对程序段1进行编译；否则对程序段2进行编译。

```c
#ifndef 标识符
程序段 1
#else
程序段 2
#endif
```

如果标识符未被`#define`命令定义过则对程序段1进行编译，反之编译程序段2.

```c
#if 常量表达式
程序段 1
#else
程序段 2
#endif
```

如果常量表达式的值为true，则对程序段1编译，反之则编译程序段2。

#### error

强制停止编译，用于编译时调试。

#### line

#### pragma