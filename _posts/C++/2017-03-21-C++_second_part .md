---
layout: post
title: C++小结（2）
category: 编程语言
keywords: C++,2017，编程语言
---

# C++第二部分小结

考试太多...耽误了一些时间

---
## 1. IO库
* 使用good或fail是确定流总体装填的正确办法。eof和bad操作只能表示特定的错误
* 如果程序异常中止，输出缓冲区是不会被刷新的
* 每个流同时最多关联到一个流，但多个流可以同时关联到同一个ostream p283

### 1.1文件流
#### 1.1.1文件模式：

|模式|介绍|
|:---:|:---:|
|in|为输入（读）打开文件|
|out|为输出（写）打开文件|
|ate|初始位置：文件尾|
|app|所有输出附加在文件末尾|
|trunc|如果文件已存在则删除文件|
|binary|二进制方式|

指定文件模式的限制：   
* out模式只可能用于ofstream或fstream
* in模式只可能用于ifstream或fstream
* 只有out被设定的时候才可使用trunc
* 只要trunc没被设定，就可设定app模式
* app模式下，即使没有显示指定out模式，文件也总以输出方式打开
* 默认情况下，即使没有制定trunc模式，以out打开的文件也会被截断
* ate和binary可用于任何类型的文件流对象并可与其他任何文件模式组合使用   

## 2. 顺序容器
### 2.1 顺序容器类型及操作

顺序容器类型|技能
---|---
vector|可变大小数组，快速随机访问。尾部之外位置插入或删除元素可能很慢
deque|双端队列，快速随机访问，头尾插入/删除很快
list|双向链表，双向顺序访问，任何位置插入/删除都超快
forward_list|单向链表，任何位置插入/删除都超快
array|固定大小数组，快速随机访问，无法添加删除元素
string|与vector相似，随机访问快，尾部插入/删除快   

**注意：复制相关运算会导致指向左边容器内部的迭代器、引用和指针失效，而swap操作将容器内容交换不会导致指向容器的迭代器、引用和指针失效（array和string除外）**

* 每个容器类型都支持相等运算符；除了无需关联容器外的所有容器都支持关系运算符。

顺序容器操作|适用类型
---|---
push_back|vector,deque,string
push_front|list,front_list,deque
insert|vector,deque,list,string,(forward_list有特殊版本)

* 删除deque中除首位位置之外的任何元素都会使所有迭代器、引用和指针失效。指向vector或string中删除点之后位置的迭代器、引用和指针都会失效

### 2.2 容器操作使迭代器失效规则
**添加元素：**
* vector或string：如存储空间被重新分配，则指向容器的迭代器、指针和引用都会失效。若未重新分配，指向插入位置之前的元素的迭代器、指针和引用仍有效，但之后元素的迭代器、指针和引用将会失效
* deque：插入到除首位位置之外的任何位置都会导致迭代器、指针和引用失效。如在首位位置添加元素，迭代器会失效，但指向存在元素的的引用和指针不会失效
* list或forward_list：真相容器的迭代器、指针和引用仍有效。   

**删除元素：**
* list或forward_list：指向容器其他位置的迭代器、引用和指针仍有效
* deque：若在首位之外的任何位置删除元素，那么指向被删除元素外其他元素的迭代器、引用或指针也会失效。若删除尾元素，除尾后迭代器外，其他迭代器、引用和至真不受影响；如果是首元素，也不会受影响
* vector和string：被删掉元素之前元素的迭代器、引用和指针仍有效
*注意：当我们删除元素时，尾后迭代器总会失效*

### 2.3迭代器循环
* 调用erase后，不必递增迭代器
* 调用insert后，需递增迭代器两次

### 2.4容器适配器
> 本质上来说，适配器是一种机制，能使某种事物的行为看起来像另外一种事物一样。   
> 例如，stack适配器接受一个顺序容器（除array或forward_list外），并使其操作起来像一个stack一样

适配器可以使用的容器种类是有限制的：
* stack（push_back、pop_back、back）：除array和forward_list之外的任何容器类型
* queue（back、push_back、front、push_front）：list或deque，不能基于vector
* priority_queue（front、push_back、pop_back、随机访问能力）：vector或deque，不能基于list

## 3.泛型算法
### 3.1只读算法
* accumulate:
```c++
int sum = accumulate(vec.cbegin(), vec.cend(), 0);//vec 为int, double , long ...
string sum = accumulate(v.cbegin(), v.cend(), string("")); //必须为string(""), 否则""类型为const char*
```
* equal:
```c++
//roster2中的元素数目至少应与roster1一样多，但容器类型、元素类型不必一样
equal(roster1.cbegin(), roster1.cend(), roster2.cbegin());
```

### 3.2写容器元素的算法
* fill，fill_n:
```c++
fill(vec.begin(), vec.end(), 0);//将所有元素置零
fill_n(vec.begin(), vec.size(), 0);//将所有元素置零
```
**注意：序列原大小至少不小于我们要求算法写入的元素数目**     

* 插入迭代器: back_inserter------<iterator.h>
```c++
vector<int> vec; //空向量
auto it = back_inserter(vec); //通过它赋值会将元素添加到vec中
*it = 42; //vec中现在有一个元素，值为42
fill_n(back_inserter(vec), 10, 0); //添加10个元素到vec
```
* 拷贝算法：copy
```c++
int a[] = {0,1,2,3,4,5,6,7,8,9};
int a2[sizeof(a1)/sizeof(*a1)];//a2与a1大小一样
//ret指向拷贝到a2的尾元素之后的位置
auto ret = copy(begin(a1), end(a1), a2); //把a1的内容拷贝给a2，ret指向拷贝到a2尾元素之后的位置
```
replace，replace_copy:
```c++
replace(ilst.begin(), ilst.end(), 0, 42)//把所有值为0的元素改为42
//ilst并不改变，ivec包含ilst的拷贝，不过ilst为0的值都变为了42
replace_copy(ilst.cbegin(), ilst.cend(), back_inserter(ivec), 0, 42);
```
**注意：传递给copy的目的序列至少要包含与输入序列一样多的元素**

### 3.3重排容器元素的算法
*目标：简化一系列儿童词汇中所用的词汇，使每个单词只出现一次*   
* 消除重复单词
```c++
void elimDups(vector<string> &words){
  //按字典序排序words，以便查找重复单词
  sort(words.begin(), words.end());
  //unique重拍输入范围，使得每个单词出现一次
  //排列在范围的前部，返回指向不重复区域之后一个位置的迭代器
  auto end_unique = unique(words.begin(), words.end());
  //使用向量操作erase删除重复单词
  words.erase(end_unique, words.end());
}
```
**unique并不真的删除任何元素，它只覆盖相邻的重复元素，使不重复元素出现在序列开始部分**

### 3.4定制操作   

* 谓词：可调用的表达式，返回结果是一个能用做条件的值
提供给sort的谓词必须在输入序列所有可能的元素之上定义一个一致的序，如：
```c++
//比较函数，用按长度排序单词
bool isShorter(const string &s1, const string &s2){
  return s1.size() < s2.size();
}
//按长度由短至长排序
sort(words.begin(), words.end(), isShorter);
//长度相同的单词维持字典序
stable_sort(words.begin(), words.end(), isShorter);
```
* lambda表达式   
一个lambda表达式表示一个可调用的代码单元。它具有一个返回类型，一个参数列表和一个函数体。与函数不同的是，它可定义在函数内部；必须使用尾置返回来指定返回类型；不能有默认参数；
lambda表达式的形式：
```c++
[capture list](parameter list) -> return type {function body}
```
**举个例子**
```c++
auto f = [] { return 42;};
cout << f() << endl; //打印42
//与isShorter函数完成相同功能的lambda
[](const string &a, const string &b)
  { return a.size() < b.size();}
//调用lambda
stable_sort(words.begin(), word.end(),
            [](const strng &a, const string &b)
              { return a.size() < b.size(;)});
```
find_if 与 find 不同的是，第三个参数是一个谓词，find_if对输入序列的每个元素调用给定的这个谓词，返回第一个使谓词返回非零值的元素   
make_plural: ``` make_plural(count, "word", "s") ```   
for_each:
```c++
for_each(wc, words.end(),
          [](const string &s){cout << s << " ";});
cout << endl;
```
完整的例子如下：
```c++
void biggies(vector<string &words, vector<string>::size_type sz){
  elimDumps(words);
  stable_sort(words.begin(), words.end(),
              [](const string &a, const string &b)
                { return a.size() < b.size();});
  auto wc = find_if(words.begin(), words.end(),
                [sz](const string &a)
                    {return a.size() >= sz;});
  auto count = words.end() - wc;
  cout << count << " " << make_plural(count, "word", "s")
        << " of length " << sz << " or longer" << endl;
  for_each(wc, words.end(),
            [](const string &s){cout << s << " ";});
  cout << endl;
}
```
**注意：应尽量避免捕获指针或引用**

### 3.5泛型算法结构   

迭代器类别|特性
---|---
输入迭代器|只读不写；单遍扫描，只能递增(istream_iterator)
输出迭代器|只写不读；单遍扫描，只能递增(ostream_iterator)
前向迭代器|可读写；多遍扫描，只能递增(forward_list)
双向迭代器|可读写；多遍扫描，递增递减
随机访问迭代器|可读写，多遍扫描，支持全部迭代器运算(array,deque,string,vector,数组指针)

## 4.关联容器
* map：key-value对；key起索引作用，value表示与索引相关联的数据
* set：只包含一个关键字，支持高效的关键字查询操作
* 允许重复关键字的容器名字都包含multi
* 不保持关键字按顺序存储的容器名字都以unordered开头，使用哈希函数来组织元素
* 关联容器的哈希值都是双向的
* 有序容器的关键字类型---P378页
* 我们通常不对关联容器使用泛型算法，

### 4.1 pair类型

创建pair
```c++
pair<T1, T2> P;
pair<T1, T2> p(v2, v2);
pair<T1, T2> p = {v1, v2};
make_pair(v1, v2);
```

### 4.2关联容器相关操作

**插入与删除**

操作|简介
---|---
c.insert(v)<br>c.emplace(args)|v是value_type类型的对象。对于map和set，只有当元素的关键字不在c时才插入元素。函数返回一个pair，包含一个迭代器，指向具有指定关键字的元素，以及一个指示插入是否成功的bool值。<br>对于multimap和multiset，总会插入给定元素，并返回一个指向新元素的迭代器。
c.insert(b, e)<br>c.insert(il)|b和e是迭代器，表示一个c::value_type类型值的范围；il是这种值的花括号列表，函数返回void
c.insert(p, v)<br>c.emplace(p, args)|类似insert(v)，但将迭代器p作为一个提示，指出从哪里开始搜索新元素应该存储的位置。返回一个迭代器，指向具有给定关键字的元素
c.erase(k)|从c中删除每个关键字为k的元素。返回一个size_type值，指出删除元素的数量
c.erase(p)|从c中删除迭代器p指定的元素。p必须指向c中的一个真实元素，不能等于c.end()。返回一个指向p之后元素的迭代器，若p指向c中的尾元素，则返回c.end()
c.erase(b, e)|删除迭代器对b和e所表示范围中的元素，返回e

**访问**   
map：
* 只能对非const的map和unordered_map使用下标操作
* 对一个map进行下标操作时，会得到一个mapped_type对象；但当解引用一个map迭代器时，会得到一个value_type对象   

操作|简介
---|---
c[k]|返回关键字为k的元素；如k不在c中，添加一个关键字为k的元素，对其进行值初始化
c.at(k)|访问关键字为k的元素，带参数检查；若k不在c中，抛出一个out_of_range异常   

set:
操作|简介
---|---
c.find(k)|返回一个迭代器，指向第一个关键字为k的元素，若k不在容器中，则返回尾后迭代器
c.count(k)|返回关键字等于k的元素的数量。对于不允许重复关键字的容器，返回值永远是0或1
c.lower_bound(k)|返回一个迭代器，指向第一个关键字不小于k的元素
c.upper_bound(k)|返回一个迭代器，指向第一个关键字大于k的元素
c.equal_range(k)|返回一个迭代器pair，表示股那件子等于k的元素的范围。若k不存在，pair的两个成员均等于c.end()
**举个例子，使用equal_range打印匹配元素**
```c++
for (auto pos = authors.equal_range(search_item);
      pos.first != pos.second; ++pos.first)
      cout << pos.first->second << endl;
```
### 无序容器
无序容器管理操作：

桶接口|
---|---
c.bucket_count()|正在使用的桶的数目
c.max_bucket_count()|容器能容纳的最多的桶的数量
c.bucket_size(n)|第n个桶中有多少个元素
c.bucket(k)|关键字为k的元素在哪个桶中
**桶迭代** |
local_iterator|可以用来访问桶中元素的迭代器类型
const_local_iterator|桶迭代器的const版本
c.begin(n)<br>c.end(n)|桶n的首元素迭代器和尾后迭代器
c.cbegin(n)<br>c.cend(n)|与前两个函数类似，但返回const_local_iterator
**哈希策略** |
c.load_factor()|每个桶的平均元素数量，返回float值
c.max_load_factor()|c试图维护的平均桶的大小，返回float值。c会在需要时添加新的桶，以使得load_factor <= max_load_factor
c.rehash(n)|重组存储，使得bucket_count >= n且bucket_count > size/max_load_factor
c.reserve(m)|重组存储，使得c可以保存n个元素且不必rehash
* 不能直接定义关键字类型为自定义类类型的无序容器

## 5.动态内存
### 5.1智能指针
指针类型| memory.h
---|---
shared_ptr|多个指针指向同一个对象
unique_ptr|独占所指向的对象
weak_ptr|伴随类，弱引用，指向shared_ptr所管理的对象

shared_ptr和unique_ptr都支持的操作|
---|---
shared_ptr<T> sp   unique_ptr<T> up |空智能指针，可以指向类型为T的对象
p.get()|返回p中保存的指针。要小心使用，若智能指针释放了其对象，返回的指针所指向的对象也就消失了
swap(p, q)   p.swap(q) |交换p和q中的指针
**shared_ptr独有的操作** |
make_shared<T>(args)| 返回一个shared_ptr, 指向一个动态分配类型为T的对象，使用args初始化此对象
shared_ptr<T>p(q)|p是shared_ptr q的拷贝：此操作会递增q中的计数器。q中的指针必须能转换为T*
p = q|p和q都是shared_ptr， 所保存的指针必须能相互转换。此操作会递减p的引用次数，递增q的引用次数；若p的引用计数变为0，则将其管理的元内存释放
p.unique()|若p.use_count()为1，返回true;否则返回false
p.use_count()|返回与p共享对象的智能指针数量；可能很慢，主要用于调试
**shared_ptr和new结合使用**
**直接举例子！**
```c++
shared_ptr<int> p1 = new int(1024);//错误，必须使用直接初始化形式
shared_ptr<int> p2(new int(42));//正确
```
定义和改变shared_ptr的其他方法|
---|---
shared_ptr<T> p(q)|p管理内置指针q所指向的对象；q必须指向new分配的内存，且能转换为T*类型
shared_ptr<T> p(u)|p从unique_ptr u那里接管了对象的所有权：将u置为空
shared_ptr<T> p(q, d)|p接管了内置指针q所指向的对象的所有权。q必须能够转换为T*类型。p将使用可调用对象d来代替delete
shared_ptr<T> p(p2, d)|p是shared_ptr p2的拷贝，唯一区别是将用可调用对象d来代替delete
p.reset()<br>p.reset(q)<br>p.reset(q, d)|若p是为唯一指向其对象的shared_ptr, reset会释放此对象。若传递了可选的参数内置指针q，会令p指向q，否则会将p置为空。若还传递了参数d，将会调用d而不是delete来释放q
**智能指针陷阱**
* 不使用相同的内置指针值初始化（或reset）多个智能指针
* 不delete get()返回的指针
* 不使用get()初始化或reset另一个智能指针
* 如果你使用get()返回的指针，记住当最后一个对应的智能指针销毁后，你的指针就变为无效了
* 如果使用的智能指针管理的资源不是new分配的内存，记住传递给他一个删除器

unique_ptr操作|
---|---
unique_ptr<T> u1<br>unique_ptr<T, D> u2|空unique_ptr，可以指向类型为T的对象，u1会使用delete来释放它的指针；u2会使用一个类型为D的可调用对象来释放它的指针
unique_ptr<T, D> u(d)|空unique_ptr，指向类型为T的对象，用类型为D的对象d来代替delete
u.release()|u放弃对指针的控制权，返回指针，并将u置为空
u.reset()<br>u.reset(q)<br>u.reset(nullptr)|释放u指向的对象<br>如果提供了内置指针q，令u指向这个对象；否则将u置为空

weak_ptr操作|
---|---
weak_ptr<T> w|指向类型为T的对象
weak_ptr<T> w(sp)|与shared_ptr sp指向相同对象的weak_ptr。T必须能转换成sp指向的类型
w = p|p可以使一个shared_ptr或一个weak_ptr。赋值后w与p共享对象
w.reset()|将w置为空
w.use_count()|与w共享对象的shared_ptr的数量
w.expired()|若w.use_count()为0，返回true，否则返回false
w.lock()|如果expired为true，返回一个空shared_ptr；否则返回一个指向w的对象的shared_ptr

### 5.2 动态数组
* 动态数组并不是数组类型，因此不能调用begin或end。也不能使用范围for语句
* 与unique_ptr不同，shared_ptr不直接支持管理动态数组。如果希望使用shared_ptr管理一个动态数组，必须提供自己定义的删除器，**举个例子**
```c++
shared_ptr<int> sp(new int[10], [](int *p) { delete [] p; });
sp.reset(); //
```
* shared_ptr未定义下标运算符，且智能指针类型不支持指针算术运算。因此为了访问数组中的元素，必须用get获取一个内置指针，然后用它来访问数组元素
**allocator类**
标准库allocator类及其算法|
---|---
allocator<T> a|定义一个名为a的allocator对象，它可以为类型为T的对象分配内存
a.allocate(n)|分配一段原始的、未构造的内存，保存n个类型为T的对象
a.deallocate(p, n)|释放从T*指针p中地址开始的内存，这块内存保存了n个类型为T的对象；p必须是一个先前由allocate返回的指针，且n必须是p创建时所要求的大小。在调用deallocate之前，用户必须对每个在这块内存中创建的对象调用destroy
a.construct(p, args)|p必须是一个类型为T*的指针，指向一块原始内存；args被传递给类型为T的构造函数，用来在p指向的内存中构造一个对象
a.destroy(p)|p为T*类型的指针，此算法对p指向的对象执行析构函数
**举个例子**
```c++
allocator<string> alloc;
auto const p = alloc.allocate(n);
auto q = p;
alloc.construct(q++, 10, 'c');
while (q != p)
  alloc.destroy(--q);
alloc.deallocate(p, n);
```

allocator类的伴随算法|
---|---
uninitialized_copy(b, e, b2)|从迭代器b和e指出的输入范围中拷贝元素到迭代器b2制定的尾构造的原始内存中。b2指向的内存必须足够大，能容纳输入序列中元素的拷贝<br>返回指向最后一个构造的元素之后的位置的指针
uninitialized_copy_n(b, n, b2)|从迭代器b指向的元素开始，拷贝n个元素到b2开始的内存中
uninitialized_fill(b, e, t)|在迭代器b和e指定的原始内存范围中创建对象，对象的值均为t的拷贝
uninitialized_fill_n(b, n, t)|从迭代器b指向的内存地址开始创建n个对象。b必须指向足够大的未构造的院士内存，能容纳给定数量的对象

### 5.3使用标准库的文本查询程序
---
*参考资料：《C++ primer 中文第五版》*
