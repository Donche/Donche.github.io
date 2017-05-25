---
layout: post
title: Python数据分析笔记（2）：pandas 入门
category: 编程语言
keywords: Python , 2017, 编程语言
---

# pandas 的数据结构
## Series
* NumPy 数组运算都会保留索引和值之间的链接
* 可以通过Python 字典来创建Series
```python
val = Series([-1.2, -1.5, -1.7], index=['two', 'four', 'five'])
```

## DataFrame
* 可被看作由Series 组成的字典
* 通过直接传入一个等长列表或NumPy 数组组成的字典来创建DataFrame
* 可手动指定列序列进行排列.``` DataFrame(data, columns=['year', 'state', 'pop']) ```
* 嵌套字典传给DataFrame 会被解释为：外层字典的键作为列，内层键作为行索引   

```python
In [57]: pop = {'Nevada': {2001: 2.4, 2002: 2.9},
   ....: 'Ohio': {2000: 1.5, 2001: 1.7, 2002: 3.6}}
In [58]: frame3 = DataFrame(pop)

In [59]: frame3
Out[59]:
      Nevada  Ohio
2000     NaN   1.5
2001     2.4   1.7
2002     2.9   3.6
```
* index 对象是不可修改的，以此保证多个数据结构之间的安全共享

index 函数|说明
---|---
append|连接另一个index对象，产生一个新的index
diff|计算差集，并得到一个index
intersection|计算交集
union|计算并集
isin|计算一个指示各值是否都包含在参数集合中的布尔型数组
delete|删除索引i 处的元素并得到新的index
drop|删除传入的值并得到新的index
insert|将元素插入到索引i 处，并得到新的index
is_monotonic|当各元素均大于等于前一个元素时，返回True
is_unique|当Index 没有重复值时，返回True
unique|计算index 中唯一的数组

# 基本功能
## 重新索引
* 使用reindex进行重新索引，可引入缺失值如：
```python
obj.reindex(['a', 'b', 'c', 'd', 'e'], fill_value=0)
```
* 或指定```method = 'ffil'```(或pad)向前值填充，或bfill(backfill)。使用limit指定最大填充量
* 还可指定```columns = xxx```重新索引列

## 丢弃指定轴上的项
```python
obj = Series(np.arange(5.), index=['a', 'b', 'c', 'd', 'e'])
obj.drop(['d', 'c'])
# 可删除任意轴上的索引值
data.drop('two', axis=1)
```

## 索引、选取、过滤
* **利用标签的切片运算其末端是包含的**

## 算术运算和数据对齐
* 可使用add 、sub、div、mul 等方法加上fill_value 设置默认填充值
* 默认情况下，DataFrame和Series之间的算术运算会将Series的索引匹配到DataFrame的列，然后沿着行一直向下广播

## 函数应用和映射
* 可用DataFrame 的apply 方法用于各行各列的一维数组操作，如
```python
In [164]: def f(x):
     ...:     return Series([x.min(), x.max()], index=['min', 'max'])
In [165]: frame.apply(f)
```
* 使用applymap(f) 实现元素级的操作。 Series 用map(f)

## 排序
* 使用```sort_index(axis = 1,ascending = False)``` 进行排序
* 使用order(by = ['a', 'b']) 按值排序，默认由低到高，NaN在末尾
* 使用rank() 为各组分配一个平均排名

# 汇总和计算描述统计
* 使用skipna=False 禁用在汇总计算中排除NA值
* 与描述统计相关的方法：
![shortCutOfPandas](https://raw.githubusercontent.com/Donche/Donche.github.io/master/_posts/Python/SC_2_1.jpg)
![shortCutOfPandas](https://raw.githubusercontent.com/Donche/Donche.github.io/master/_posts/Python/SC_2_2.jpg)
* Series 中的 corr 用于计算两个Series 中重叠的、非NA的、按索引对其的值的相关系数。cov用于计算协方差
* DataFrame 中的corr 和cov 用于返回完整的相关系数和协方差矩阵。用corrwith 计算与另一个Series 或DataFrame 之间的相关系数
* isin：计算一个表示“Series 各值是否包含于传入的值序列中” 的布尔型数组
* unique： 计算Series 中的惟一值数组，按发现的顺序返回
* value_counts：返回一个Series，其索引为惟一值，值为频率，按计数值降序排列

# 处理缺失数据
* dropna：根据各标签的值是否存在缺失数据对轴标签进行过滤，可通过阈值调节对缺失值的容忍度
* fillna：用指定值或插值方法填充缺失数据
* isnull：返回含有布尔值对象，表示哪些是缺失值
* notnull：isnull的否定式

* dropna默认丢弃任何含有缺失值的行。可用how="all" 只丢弃全为NA的行。丢弃列用axis=1。thresh用来设定需要保留数据的最小个数
* 使用字典调用fillna 可实现对不同的列填充不同的值。inplace=True 表示修改调用者对象而不产生副本

# 层次化索引
* 一个Series 可通过unstack方法被安排到一个DataFrame中。逆运算为stack
* wswaplevel接受两个级别的编号或名称，返回一个互换了级别的新对象，但数据不会变化。随后使用sortlevel排序。
* 使用DataFrame 的set_index 函数将其一个或多个列转换为行索引，并创建一个新的DataFrame。默认这些列会被移除，但也可使用drop=False 将其保留。
* 使用reset_index 将层次化索引级别转移到列里面。

---
*参考资料：《利用Python 进行数据分析》*
