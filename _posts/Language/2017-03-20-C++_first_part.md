---
layout: post
title: C++小结（1）
category: 编程语言
keys: 
  - C++
  - 2017
  - 编程语言
---
# C++第一部分小结
仅关于《C++ primer 中文第五版》第一部分 “C++基础” 的一些个人易混点的总结。待完善...
* [1. 变量及其声明](#1)
  * [1.1 变量的初始化](#1.1)
  * [1.2 顶层const与底层const](#1.2)
  * [1.3 const与constexpr](#1.3)
  * [1.4 auto与deltype](#1.4)
* [2.字符串、向量及数组](#2)
  * [2.1 迭代器](#2.1)
* [3.表达式](#3)
  * [3.1 类型转换](#3.1)
* [4.语句](#4)
* [5.函数](#5)
  * [5.1 函数重载](#5.1)
  * [5.2 函数指针](#5.2)
* [6.类](#6)
  * [6.1 聚合类](#6.1)
  * [6.2 字面值常量类](#6.2)


<span id="1"></span>
## 1. 变量及其声明

<span id="1.1"></span>
### 1.1 变量的初始化:
* 引用不是一个对象
* 引用必须被初始化
* 指针的值必须处于以下四种状态之一：指向一个对象、指向对象所占空间的下一个位置、空指针、无效指针。  

<span id="1.2"></span>  
### 1.2 顶层const与底层const
* 顶层const (top-level const): 指任意的对象是常量，对任意数据类型都适用。
* 底层const (low-level const): 指针所指对象是个常量。与指针、引用等复合类型的基本类型有关。
如书中P57例子:

```c++
int i = 0;
int *const p1 = &i;     //不能改变p1的值，这是一个顶层const
const int ci = 42;      //不能改变ci的值，这是一个顶层const
const int *p2 = &ci;    //允许改变p2的值，这是一个底层const
const int *const p3 = p2;   //靠右的const是顶层const，靠左的是底层const
const int &r = ci;  //用于声明引用的const都是底层const
```

当进行对象拷贝操作时，顶层const不会受影响，而底层const的限制则不可忽视。拷贝时，拷入和拷出对象必须有相同的底层const资格，或可以转换（如非常量一般可以转化为常量，反之则不行）。**举个例子：**
```c++
int *p = p3;  //错误：p3有底层const而p没有
p2 = p3;   //正确：均为底层const
p2 = &i;   //正确：int* 可以转换为 const int*
int &r = ci;  //错误：普通的int&不能绑定到int常量上
const int &r2 = i;   //正确：const int& 可以绑定到一个普通int上
```
<span id="1.3"></span>
### 1.3 const与constexpr
首先需要清楚几个概念:  
**常量表达式：** 值不会改变并且在编辑过程中就能得到结果的表达式  
**字面值类型：** 类型简单，值显而易见的类型，如0为int型，1.1为double型  
*&diams;目前为止接触过的类型中，算数类型、引用和指针都属于字面值类型*  
*&diams;当一个指针为constexpr时，初始值必须为nullptr、0 或存储于某个固定地址的对象，且constexpr仅对指针有效，与指针所指对象无关，即constexpr会将所定义对象置为顶层const*  

**举个例子：**
```c++
const int *p = nullptr;  //p为指向整型常量的指针
constexpr int *q = nullptr;  //q为指向整型的常量指针
```
<span id="1.4"></span>
### 1.4 auto与deltype
**auto:**  
* 定义的变量必须有初始值。如在一条语句中声明多个变量时，该语句所有变量的基本数据类型必须一样  
* 当引用被用作初始值时，编译器会以引用对象的类型作为auto的类型  
* auto一般会忽略掉顶层const，但底层const会被保留下来。但当类型为auto的引用则会将初始值的顶层const保留    

**decltype：**
* 分析表达式类型，并不计算表达式的值
* 如表达式为一个变量，decltype返回变量的类型（包括顶层const及引用）  
* 如表达式内容是解引用操作，则decltype得到引用类型
* 如decltype使用的是一个不加括号的变量，则得到的结果是该变量的类型；如给变量加上了一层或多层括号，编译器会将其当成一个表达式，从而得到引用类型。

**注意：引用从来都作为其所指对象的同义词出现，只有decltype是一个例外**

**举一个例子**
```c++
int i = 42, *p = &i, &r = i;
decltype(r + 0) b; //正确：假发的结果是int，因此b是一个未初始化的int
decltype(*p) c; //错误：c是int&，必须初始化
decltype((i)) d; //错误：d是int,必须初始化
decltype(i) e; //正确：e是一个（未初始化的）int
```

<span id="2"></span>
## 2. 字符串、向量及数组
<span id="2.1"></span>
### 2.1 迭代器
const_iterator: 可以读取但不可以修改它所指的元素值。如对象是常量，则只能使用const_iterator   
**注意：迭代器之间可以相减，但不可以相加**
<span id="3"></span>
## 3. 表达式
<span id="3.1"></span>
### 3.1 类型转换
#### 3.1.1 隐式转换
**算数转换** 规则：  
1.整型提升  
2.如一个是无符号类型，另外一个是带符号的，其中无符号类型不小于带符号，则带符号转换成无符号的  
3.带符号大于无符号时，则结果依赖于机器。若无符号类型的所有值都能储存在带符号类型中，则无符号类型转换为带符号类型；如果不能则相反。

其他隐式类型转化  
数组->指针：当数组被用作decltype的参数、作为取地址符（&），sizeof以及typeid等运算符对象、或用引用来初始化数组的时候，转换不会发生。
#### 3.1.2 显式转换
static_cast:任何具有明确定义的类型转换，只要不包含底层const，都可以使用     
dynamic_cast：支持运行时类型识别，后续再讲     
const_cast：只能改变运算对象的底层const。若对象本身不是常量，则通过强制类型转换获得写权限是合法的行为。否则会产生未定义的结果。常用于有函数重载的上下文中。      
reinterpret_cast：通常为运算对象的位模式提供较低层次上的重新解释。应尽量避免使用。
**例如**
```c++
const char *cp;
char *q = static_cast<char*>(cp); //错误：static_cast不能转换掉const性质
static_cast<string>(cp); //正确：字符串值转换成string类型
const_cast<string>(cp); //错误：const_cast只改变常量属性
```
**注意：强制类型转换干扰了正常类型检查，应避免使用。**
<span id="4"></span>
## 4. 语句
<span id="4.1"></span>
### 4.1 try语句块和异常处理
try语句块的通用语法形式为
```c++
try {
  program-statements
} catch (exception-declaration) {
  handler-statements
} catch (exception-declaration) {
  handler-statements
} //...
```
当异常被抛出时，首先搜索抛出该异常的函数。如过没找到匹配的catch子句，则终止函数，并在调用该函数的函数中继续寻找。以此类推。如果最终还是没能找到任何匹配的catch子句，程序转到名为terminate的标准库函数。一般来说，执行这个函数会导致程序非正常退出。*如果没有try语句的异常，系统直接调用terminate函数并终止当前程序的执行*
<span id="5"></span>
## 5. 函数
<span id="5.1"></span>
### 5.1 函数重载
顶层const不影响传入函数的对象。一个拥有顶层const的形参无法和另一个没有顶层const的形参区分开来。**举个例子**
```c++
Record lookup(Phone);
Record lookup(const Phone); //重复声明了Record lookup(Phone)

Record lookup(Phone*);
Record lookup(Phone* const); //重复声明了Record lookup(Phone*)

Record lookup(Account&); //函数作用于Account的引用
Record lookup(const Account&); //新函数，作用于常量引用

Record lookup(Account*); //函数作用于指向Account的指针
Record lookup(const Account*); //新函数，作用于指向常量的指针
```
**注意：main函数不能重载**
<span id="5.2"></span>
### 5.2 函数指针
函数指针指向函数，形如下：
```c++
bool lengthCompare(const string &, const string &);
bool (*pf)(const string &, const string &); //未初始化
pf = lengthCompare;
pf = &lengthCompare; //与上句等价

bool b1 = pf("hello", "goodbye"); //调用lengthCompare函数
bool b1 = (*pf)("hello", "goodbye"); //等价的调用
bool b1 = lengthCompare("hello", "goodbye"); //等价的调用
```

**(\*pf)两端括号必不可少**  
当我们使用重载函数时，上下文必须清晰地界定到底使用哪个函数。函数指针类型必须与重载函数中的一个精确匹配。  

**函数指针形参:**  
直接 **举例子** 吧..
```c++
//以下为两个等价的声明
void usrBigger(const string &si, const string &s2,
                bool pf(const string &, const string &));
void usrBigger(const string &si, const string &s2,
                bool (* pf)(const string &, const string &));
//使用时直接把函数当实参使用即可，会自动转换成指针
useBigger(s1, s2, lengthCompare);
```
我们还可以使用类型别名和decltype简化代码

**返回指向函数的指针**     
**注意**：返回类型不会自动转换为指针，需显式将返回类型指定为指针，举个例子：
```c++
using F = int(int*, int); //F 是函数类型，不是指针
using PF = int(*)(int*, int); //PF 是指针类型

PF f1(int); //正确：PF是指向函数的指针，f1返回指向函数的指针
F f1(int);  //错误：F是函数类型，f1不能返回一个函数
F *f1(int);  //正确：显式地指定返回类型是指向函数的指针
//或使用下面的形式直接声明f1
int (*f1(int))(int*, int);
```
以上还可采用尾置返回类型的方式：```auto fl(int) -> int(*)(int*, int); ```

**将auto和decltype用于函数指针类型**      
如果我们明确知道返回的函数是哪一个，就可以使用decltype简化书写函数指针返回类型的过程。但需注意，decltype返回函数类型而非指针类型，因此需显示加上*以表明我们需要返回指针。**举个例子**
```c++
string::size_type sumLength(const string&, const string&);
string::size_type largerLength(const string&, const string&);
//根据其形参的取值，getFcn函数返回指向sumLength或者largerLength的指针
decltype(sumLength) *getFcn(const string &);
```
<span id="6"></span>
## 6. 类
<span id="6.1"></span>
### 6.1 聚合类
聚合类使得用户可以直接访问其成员，并且具有特殊的初始化语法形式。需满足如下条件：
* 所有成员都是public的
* 没有定义人和构造函数
* 没有类内初始值
* 没有基类，也没有virtual函数   

<span id="6.2"></span>
### 6.2 字面值常量类
&diams;数据成员都是字面值类型的聚合类是字面值常量类  
&diams;如不是聚合类，但若符合下述要求，则也是字面值常量类：
* 数据成员都必须是字面值类型
* 类必须至少含有一个constexpr构造函数
* 如果一个数据成员含有类内初始值，则内置类型成员的初始值必须是一条常量表达式；或者如果成员属于某种类类型，则初始值必须使用成员自己的constexpr构造函数
* 类必须使用析构函数的默认定义，该成员负责销毁类的对象  
事实上，一个字面值常量类必须至少提供一个constexpr构造函数，可声明成=default的形式。*constexpr构造函数体一般来说应该是空的。*，且必须初始化所有数据成员

---
*参考资料：《C++ primer 中文第五版》*
