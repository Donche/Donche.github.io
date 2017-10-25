---
layout: post
title: C++小结（3）
category: 编程语言
tags: 
  - 2017
  - C++
  - 编程语言
---

# C++第三部分小结

终于，第三部分也来了...

## 1.拷贝控制
### 1.1拷贝、赋值与销毁
**拷贝初始化**
* 拷贝初始化会在我们用=定义变量时发生，以下情况也会发生
* 将一个对象作为一个实参传递给一个非引用类型的形参
* 从一个返回类型为非引用类型的函数返回一个对象
* 用花括号列表初始化一个数组中的元素或一个聚合类中的成员

* 当我们出书画标准库容器或调用其insert或push成员时，容器会对其元素进行拷贝初始化
* 用emplace成员创建的元素都进行直接初始化

**拷贝赋值运算符**
* 赋值运算符通常应该返回一个指向其左侧运算对象的引用
*举个例子*
```c++
Sales_data& Sales_data::operator+(const Sales_data &rhs){
  bookNo = rhs.bookNo;
  units_sold = rhs.units_sold;
  revenue = rhs.revenue;
  return * this; //返回一个此对象的引用
}
```
**析构函数**
* 当指向一个对象的引用或指针离开作用域时，析构函数不会执行
* 成员实在析构函数体之后隐含的析构阶段中被销毁的。在整个对象销毁过程中，析构函数体是作为成员销毁步骤之外的另一部分而进行的
**三/五法则**
* 通常情况下，如果一个类需要一个析构函数，我们几乎可以肯定它也需要一个拷贝构造函数和一个拷贝赋值运算符
* 如果一个类需要一个拷贝构造函数，那它也需要一个拷贝赋值运算符，反之亦然。然而并不必然意味着也必须析构函数
**阻止拷贝**
* 我们可以通过将拷贝构造函数和拷贝赋值运算符定义为删除的函数来组织拷贝。例如：
```c++
struct NoCopy{
  NoCopy () = default;
  NoCopy (const NoCopy&) = delete; //阻止拷贝
  NoCopy &operator = (const NoCopy&) = delete; //阻止赋值
  ~NoCopy () = default;
  //其他成员
};
```
* 我们可以对人和函数指定 = delete
* 析构函数不能被delete
```c++
struct NoDtor {
  NoDtor () = default;
  ~NoDtor () = delete;
};
NoDtor nd; //错误：NoDtor的析构函数是删除的
NoDtor *p = new NoDtor();//正确：但我们不能delete p
delete p; //错误：NoDtor的析构函数是删除的
```
**合成的拷贝控制成员可能是删除的**
* 如果类的某个成员的析构函数是删除的或不可访问的（如是private的），则类的合成析构函数被定义为删除的
* 如果类的某个成员的拷贝构造函数是删除的或不可访问的，则类的合成拷贝构造函数被定义为删除的。如果类的某个成员的析构函数是删除的或不可访问的，则类合成的拷贝构造函数也被定义为删除的。
* 如果类的某个成员的拷贝赋值运算符是删除的或不可访问的，或有类有一个const的或引用成员，则类的合成拷贝赋值运算符被定义为删除的。
* 如果类的某个成员的析构函数是删除的或不可访问的，或是类有一个引用成员，它没有类内初始化器，或是类有一个const成员，它没有类内初始化器且其类型为显示定义默认构造函数，则该类的默认构造函数被定义为删除的
**本质上，这些规则的含义是，如果一个类有数据成员不能默认构造、拷贝、复制或销毁，则对应的成员函数将被定义为删除的**
* 希望组织拷贝的类应该使用 =delete 来定义他们自己的拷贝构造函数和拷贝复制运算符，而不应该将他们声明为private的

### 1.2拷贝控制和资源管理
* 确定对象的拷贝语义一般有两种选择，通过定义拷贝操作，是类的习惯为看起来像一个值或者像一个指针。

### 1.3交换操作
使用拷贝和交换的赋值运算符自动就是异常安全的，且能正确处理自赋值，如
```c++
HasPtr& HasPtr::operator = (HasPtr rhs){
  swap(* this, rhs);  //rhs现在指向本对象曾经使用的内存
  return * this; //rhs被销毁，从而delete了rhs中的指针
}
```

* 标准库容器，string和shared_ptr既支持移动也支持拷贝。IO类和unique_ptr可以移动但不能拷贝

### 1.3右值引用

* 一般而言，一个左值表达式表达的是一个对象的身份，而一个右值表达式表示的是对象的值
* 由于右值引用只能绑定到临时对象，我们得知：1-所引用的对象将要被销毁，2-该对象没有其他用户（真是一个伤感的数据类型...）
* 将一个左值显性转换为右值：``` int &&rr3 = std::move(rr1); ```但调用move后意味着我们除了对rr1赋值或销毁外不能再使用它，即我们不能对移动后源对象的值做任何假设

* 只有当一个类没有定义任何自己版本的拷贝控制成员，且它的所有数据成员都能移动构造或移动赋值时，编译器才会为它合成移动构造函数或移动赋值运算符

## 2. 重载运算与类型转换

### 基本概念与设计准则
* 通常情况下，不应该重载逗号、取地址、逻辑与和逻辑或运算符
* 赋值(=)、下标([])、调用(())和成员访问箭头(->)必须是成员。
* 复合赋值运算符一般来说是成员，但并非必须，这一点与赋值运算符略有不同。
* 改变对象状态的运算符或者与给定类型密切相关的运算符，如递增、递减和解引用运算符，通常是成员。
* 具有对称性的运算符可能转换任意一端的运算对象，例如算数、相等性、关系和位运算等等，因此他们通常应该是普通的非成员函数
* 通常情况下，IO运算符会被声明成友元
* 输入运算符必须处理输入可能失败的情况，而输出运算符不需要
* 如果类同时定义了算术运算符和相关的复合赋值运算符，则通常情况下应该使用复合赋值来实现算术运算符
一些关于相等运算符的设计准则：
* 如果一个类含有判断两个对象是否相等的操作，则它显然应该把函数定义成operator==而非一个普通的命名函数：因为用户肯定希望能使用==比较对象，而且定义==之后也更容易使用标准库容器和算法
* 如果类定义了operator==，则该运算符应该能判断一组给定的对象中是否含有重复数据
* 通常情况下，相等运算符应该具有传递性
* 如果类定义了operator==，则也应该定义operator！=
* 相等运算符和不相等运算符中的一个应该把工作委托给另外一个。

## 3.面向对象的程序设计
* 基类通常都应该定义一个虚析构函数，即使该函数不执行任何实际操作也是如此
* 派生类对象不能直接初始化基类的成员，尽管从语法上来说我们可以在派生类构造函数体内给他的共有或受保护的基类成员赋值，但最好不要这么做
* 如基类定义了一个静态成员，则在整个继承体系中只存在该成员的唯一定义
* 派生类向基类的自动类型转换只对指针或引用类型有效
* 重构负责重新设计类的体系以便将操作和/或数据从一个类移动到另一个类中
* 派生类的成员和友元只能访问派生类对象中的基类部分的受保护成员，对普通基类对象中的成员不具有特殊的访问权限。 **举个例子**
```c++
class Base {
protected:
  int prot_mem;     //protected成员
};
class Sneaky : public Base {
  friend void clobber (Sneaky&);    //能访问Sneaky::prot_mem
  friend void clobber(Base&);   //不能访问Base::prot_mem
  int j;  //j默认是private
}；
//正确： clobber能访问Sneaky对象的private和protected成员
void clobber(Sneaker &s) {s.j = s.prot_mem = 0;}
//错误：clobber不能访问base的protected成员
void clobber(Base &b) {b.prot_mem = 0;}
```

### 派生类向基类转换的可访问性
* 只有当D公有地继承B时，用户代码才能使用派生类向基类的转换；如果D继承B的方式是受保护的或者私有的，则用户代码不能使用此转换
* 不论D以什么方式继承B,D的成员函数和友元都能使用派生类向基类的转换；派生类向其直接基类的类型转换对于派生类的成员和友元来说永远是可访问的
* 如果D继承B的方式是公有的或者受保护的，则D的派生类的成员和友元可以使用D向B的类型转换；反之，如果D继承B的方式是私有的，则不能使用

### 构造函数与拷贝控制
* 如果一个类定义了析构函数，即使它通过=default的形式使用了合成的版本，编译器也不会为这个类合成移动操作
* 一旦一个类定义了自己的移动操作，那么它必须同时显式地定义拷贝操作
* 通常情况下，using声明语句只是令某个名字在当前作用域内可见。而当作用于构造函数时，using声明语句将令编译器产生代码
* 一个构造函数的using声明不会改变该构造函数的访问级别，而且不能指定explicit或constexpr，而且默认、拷贝和移动构造函数不会被继承


## 模板与泛型编程
### 模板函数
* 非类型模板参数表示一个值而非一个类型，且实参必须是常量表达式。**举个例子**
```c++
template<unsigned N, unsigned M>
int compare(const char (&p1)[N], const char (&p2)[M]){
  return strcmp(p1, p2);
}
```
* inline或constexpr说明符应放在模板参数列表之后，返回类型之前
* 与非模板不同，模板的头文件通常既包括声明也包括定义

### 模板类
* 当我们使用一个类模板时必须提供模板实参，但这一规则有一个例外。在类模板自己的作用域中，我们可以直接使用模板名而不提供实参
* 如果一个类模板包含一个非模板友元，则友元被授权可以访问所有的模板实例。如果友元自身是模板，则类可以授权给所有友元模板实例，也可以只授权给特定实例
**类模板与友元**
* 一对一友好关系

```c++
template <typename> class BlobPtr;
template <typename> class Blob;
template <typename T>
  bool operator==(const Blob<T>&, const Blob<T>&);

template <typename T> class Blob {
  //每个Blob实例将访问权限授予用相同类型实例化的BlobPtr和相等运算符
  friend class BlobPtr<T>;
  friend bool operator==<T>  (const Blob<T>&, const Blob<T>&);
  //.....;
};
```
* 通用和特定的模板友好关系
```c++
//前置声明，在将模板的一个特定实例声明为友元时要用到
template <typename T> class Pal;
class C { //C是一个普通的非模板类
    friend class Pal<C>; //用类C实例化的Pal是C的一个友元
    //Pal2的所有实例都是C的友元；这种情况无需前置声明
    template <typename T> friend class Pal2;
};
template <typename T> class C2 {
  //C2的每个实例都将相同实例化的Pal声明为友元
  friend class Pal<T>; //Pal的模板声明必须在作用域之类
  //Pal2的所有实例都是C2的每个实例的友元，不需要前置声明
  template <typename X> friend class Pal2;
  //Pal3是一个非模板类，它是C2所有实例的友元
  friend class Pal3; //不需要Pal3的前置声明
};
```
* 默认模板实参
```c++
template <typename T, typename F = less<T>>
int compare(const T &v1, const T &v2, F f = F()){
  if(f(v1, v2)) return -1;
  if(f(v2, v1)) return 1;
  return 0;
}
```

* 控制实例化
显式实例化：
```c++
extern template class Blob<string>;//声明
template int compare(const int&, const int&);//定义
```
而模板的定义必须出现在程序的其他文件中：
```c++
//templateBuild.cc
//实例化文件必须为每个在其他文件中声明为extern的类型和函数提供一个（非extern）的定义
template int compare(const int&, const int&);
template class Blob<string>;//实例化类模板的所有成员
```
* 在一个类模板的实例化定义中，所有类型必须能用于模板的所有成员函数。因为，与处理类模板的普通实例化不同，编译器会实例化该类的所有成员。
* 通过在编译时绑定删除器，unique_ptr避免了间接调用删除器的运行时开销。通过在运行时绑定删除器，shared_ptr使用户重载删除器更为方便。

### 模板实参推断
* 编译器通常不是对实参进行类型转换，而是生成一个新的模板实例
* 能在调用中应用于函数模板的有如下两项：const转换、数组或函数指针的转换。其他类型转换，如算数转换、派生类向基类的转换以及用户定义的转换都不能应用于函数模板
* 显式模板实参例子：

```c++
template <typename T1, typename T2, typename T3>
T1 sum(T2, T3);
//T1是显示指定的，T2和T3是从函数实参类型推断而来的
auto val3 = sum<long long>(i, lng); //long long sum(int, long)
```
* 对于模板类型参数已经显示指定了的函数实参，也进行正常的类型转换。
* 尾置返回类型允许我们在参数列表之后声明返回类型，例如：
```c++
template <typename It>
auto fcn(It beg, It end) -> decltype(* beg){
  //处理序列
  return * beg;//返回序列中一个元素的引用
}
```
* 在函数中返回元素值的拷贝
```c++
//为了使用模板参数成员，必须使用typenae
template <typename It>
auto fcn2(It beg, It end) ->
  typenae remove_reference<decltype(* beg)> :: type{
    //处理序列
    return * beg;//返回序列中一个元素的拷贝
  }
```
* 函数指针   

```c++
template <typename T> int compare(const T&, const T&);
//pf1指向实例int compare(const int&, const int&)
int (*pf1) (const int&, const int&) = compare;

void func(int( * )(const string&, const string&));
void func(int( * )(const int&, const int&));
func(compare); //错误
func(compare<int>); //正确
```
* 引用折叠和右值引用参数
  * 当我们将一个左值转嘀给函数的右值引用参数，且此右值引用指向模板类型参数（如T&&）时，编译器推断模板类型参数为实参的左值引用类型。因此当我们调用f3(i)时，编译器推断T的类型为int&，而非int
  * 如果我们间接创建一个引用的引用，则这些引用形成了折叠。即对于一个给定类型X：
  X& &、X& &&和X&& &都折叠成X&， 而X&& &&折叠成X&&
* 可以通过static_cast显式得将一个左值转换为一个右值引用
* 如果一个函数参数是指向模板类型参数的右值引用（如T&&），它对应的实参的const属性和左值/右值属性将得到保持

### 函数转发
* 如果实参是一个右值，则Type是一个普通（非引用）类型，forward<Type>将返回Type&&。
* 如果实参是一个左值，则通过引用折叠，Type本身是一个左值引用类型。在此情况下，返回类型是一个指向左值引用类型的右值引用，再次对forward<Type>的返回类型进行引用折叠，将返回一个左值引用类型
* 使用forward完成翻转函数
```c++
template <typename F, typename T1, typename T2>
void flip(F f, T1 &&t1, T2 &&t2){
  f(std::forward<T2>(t2), std::forward<T1>(t1));
}
```

* 当有多个重载模板对一个调用提供同样好的匹配时，应选择最特例化的版本
* 对于一个调用，如果一个非函数模板与一个函数模板提供同样好的匹配，则选择非模版版本

### 可变参数模板
* 模板参数包： 零个或多个模板参数
* 函数参数包： 零个或多个函数参数
* 在函数参数列表中，如果一个参数的类型是一个模板参数包，则此参数也是一个函数参数包。**举个例子**

```c++
//Args是一个模板参数包；rest是一个函数参数包
//Args表示领个或多个模板类型参数
//rest表示零个或多个函数参数
template <typename T, typename... Args>
void foo(const T &t, const Args& ... rest);

template<typename ... Args> void g(Args ... args) {
  cout << sizeof...(Args) << endl;//类型参数的数目
  cout << sizeof...(args) << endl;//函数参数的数目
}
```

### 特例化
* 特例化的本质是实例化一个模板，而非重载它。因此，特例化不影响函数匹配
* 模板及其特例化版本应该声明在同一个头文件中。所有同名模板的声明应该放在前面，然后是这些模板的特例化版本
* 我们只能部分特例化类模板，而不能部分特例化函数模板
```c++
template <typename T> struct Foo {
  Foo(const T &t = T()) : mem(t) { }
  void Bar() {/ * ... * /}
  T mem;
  // 其他成员
};
template<>
void Foo<int>::Bar(){
  //int的特例化处理
}
```
---
*参考资料：《C++ primer 中文第五版》*
