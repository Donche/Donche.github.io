---
layout: post
title: Python数据分析笔记（4）：Matplotlib 入门、数据的聚合与分组
category: 编程语言
keywords: Python , 2017, 编程语言
---

# matplotlib API 入门
画图逻辑跟MATLAB非常相似，所以jian'dan'de
```python
import matplotlib.pyplot as plt
fig = plt.figure()
ax1 = fig.add_subplot(2, 2, 1)
ax2 = fig.add_subplot(2, 2, 2)
ax3 = fig.add_subplot(2, 2, 3)

from numpy.random import randn
plt.plot(randn(50).cumsum(), 'k--')
_ = ax1.hist(randn(100), bins=20, color='k', alpha=0.3)
ax2.scatter(np.arange(30), np.arang(30) + 3 * randn(30))


# or create subplots like this:
fig, axes = plt.subplots(2, 3, sharex = True, sharey = True) #Using the same x and y axes

# adjust:
subplots_adjust(left=None, bottom=None, right=None, top=None, wspace=None, hspace=None)
# colors and marks:
plot(randn(30).cumsum(), color='k', linestyle='dashed', marker='o', drawstyle='steps=post')
# or:plt.plot(randn(30).cumsum(), 'ko--')
plt.legend(loc='best')
plt.xlim([0,10])
# or :ax.set_xlim([0,10])
ticks = ax.set_xticks([0, 250, 500, 750, 100])
labels = ax.set_xticklabels(['one', 'two', 'three', 'four', 'five'], rotation=30, fontsize='small')
ax.set_title('Some Plot')
ax.set_xlabel('stages')
# add annotations
ax.annotate(label, xy=(date, spx.asof(data)+50), xytext=(data, spx.asof(data)+200), arrowprops=dict(facecolor='black'), horizontalalignement='left', verticalalignement='top')

# add some figures
rect = plt.Rectangle((0.2, 0.75), 0.4, 0.15, color='k', alpha=0.3)
circ = plt.Circle((0.7, 0.2), 0.15, color='b', alpha=0.3)
ax.add_patch(rect)
ax.add_patch(circ)

# save figure:
plt.savefig('filename.png', dpi=400, bbox_inches='tight', facecolor='w')
```

# 数据聚合与分组
## GroupBy 技术
* 拆分-应用-合并（split - apply - combine）

```python
#按key1、key2进行分组，并计算data1列的平均值
means = df['data1'].groupby(df['key1'], df['key2']).mean()
#或简写为：
means = df.groupby(df['key1'], df['key2'])['data1'].mean()
#其实，分组键可以是任意长度适当的数组
states = np.array(['Ohio', 'California', 'California', 'Ohio', 'Ohio'])
years = np.array([2005, 2005, 2006, 2005, 2006])
df['data1'].groupby([states, years]).mean()
```
### 对分组进行迭代
GroupBy对象支持迭代，可以产生一组二元元组（由分组名和数据块组成）。例如：
```python
for name, group in df.groupby('key1'):
  #...
```

groupby默认是在axis=0上进行分组的，通过设置也可以在其他任何轴上进行分组。例如：
```python
grouped = df.groupby(df.dtypes, axis=1)
```
### 通过字典或Series进行分组
直接举例子吧...一目了然：
```python
In [40]:people
Out[40]:
                a            b          c         d         e
Joe      1.007189    -1.296221   0.274992  0.228913  1.352917
Steve    0.886429    -2.001637  -0.371843  1.669025 -0.438570
Wes     -0.539741          NaN        NaN -1.021228 -0.577087
Jim      0.124121     0.302614   0.523772  0.000940  1.343810
Travis  -0.713544    -0.831154  -2.370232 -1.860761 -0.860757
# 用Series也是可以的
In [41]: mapping = {'a': 'red', 'b': 'red', 'c': 'blue',
    ...:       'd': 'blue', 'e': 'red', 'f' : 'orange'}
In [42]: by_column = people.groupby(mapping, axis=1)
In [43]: by_column.sum()
Out[43]:
            blue       red
Joe     0.503905  1.063885
Steve   1.297183 -1.553778
Wes    -1.021228 -1.116829
Jim     0.524712  1.770545
Travis -4.230992 -2.405455
```

* 还可通过索引级别分组，如下：
```python
columns = pd.MultiIndex.from_arrays([['US', 'US', 'US', 'JP', 'JP'],  [1, 3, 5, 1, 3]], names=['cty', 'tenor'])
hier_df = DataFrame(np.random.randn(4, 5), columns=columns)
hier_df.groupby(level='cty', axis=1).count()
```

## 数据聚合
* 指的是任何能够从数组产生标量值的数据转换过程，例如mean, count, min, median, std, var, prod(积) sum, first, last 等。例如quantile可以计算Series或DataFrame列的样本分位数

* 若要使用自己的聚合函数，将其传入aggregate或agg:

```python
In [57]: def peak_to_peak(arr):
    ...:     return arr.max() - arr.min()

In [58]: grouped.agg(peak_to_peak)
Out[58]:
         data1     data2
key1
a     2.170488  1.300498
b     0.036292  0.487276
```

### 面向列的多函数应用

```python
grouped_pct.agg(['mean', 'std', peak_to_peak])
grouped_pct.agg([('foo', 'mean'), ('bar', np.std)])
In [68]: functions = ['count', 'mean', 'max']

In [69]: result = grouped['tip_pct', 'total_bill'].agg(functions)

In [70]: result
Out[70]:
               tip_pct                      total_bill
                 count      mean       max       count       mean    max
sex    smoker
Female False        54  0.156921  0.252672          54  18.105185  35.83
       True         33  0.182150  0.416667          33  17.977879  44.30
Male   False        97  0.160669  0.291990          97  19.791237  48.33
       True         60  0.152771  0.710345          60  22.284500  50.81
```

### 分组级运算和转换
#### Transform:

```python
In [81]: key = ['one', 'two', 'one', 'two', 'one']

In [82]: people.groupby(key).mean()
Out[82]:
            a         b         c         d         e
one -0.082032 -1.063687 -1.047620 -0.884358 -0.028309
two  0.505275 -0.849512  0.075965  0.834983  0.452620

In [83]: people.groupby(key).transform(np.mean)
Out[83]:
               a          b         c         d         e
Joe     -0.082032 -1.063687 -1.047620 -0.884358 -0.028309
Steve    0.505275 -0.849512  0.075965  0.834983  0.452620
Wes     -0.082032 -1.063687 -1.047620 -0.884358 -0.028309
Jim      0.505275 -0.849512  0.075965  0.834983  0.452620
Travis  -0.082032 -1.063687 -1.047620 -0.884358 -0.028309
```

#### apply:
* 是最一般化的GroupBy 方法，会将待处理的对象拆分成多个片段，然后对个片段调用传入的函数，最后尝试将个片段组合到一起。

```python
In [88]: def top(df, n=5, column='tip_pct'):
    ...:     return df.sort_index(by=column)[-n:]

In [89]: top(tips, n=6)
Out[89]:
     total_bill   tip     sex smoker  day    time  size   tip_pct
109       14.31  4.00  Female   True  Sat  Dinner     2  0.279525
183       23.17  6.50    Male   True  Sun  Dinner     4  0.280535
232       11.61  3.39    Male  False  Sat  Dinner     2  0.291990
67         3.07  1.00  Female   True  Sat  Dinner     1  0.325733
178        9.60  4.00  Female   True  Sun  Dinner     2  0.416667
172        7.25  5.15    Male   True  Sun  Dinner     2  0.710345
In [90]: tips.groupby('smoker').apply(top)
Out[90]:
            total_bill   tip     sex smoker   day    time  size   tip_pct
smoker
No     88        24.71  5.85    Male  False  Thur   Lunch     2  0.236746
       185       20.69  5.00    Male  False   Sun  Dinner     5  0.241663
       51        10.29  2.60  Female  False   Sun  Dinner     2  0.252672
       149        7.51  2.00    Male  False  Thur   Lunch     2  0.266312
       232       11.61  3.39    Male  False   Sat  Dinner     2  0.291990
Yes    109       14.31  4.00  Female   True   Sat  Dinner     2  0.279525
       183       23.17  6.50    Male   True   Sun  Dinner     4  0.280535
       67         3.07  1.00  Female   True   Sat  Dinner     1  0.325733
       178        9.60  4.00  Female   True   Sun  Dinner     2  0.416667
       172        7.25  5.15    Male   True   Sun  Dinner     2  0.710345
# 如果传给apply的函数能够接受其他参数或关键字，则可以将这些内容放在函数名后面一并传入
In [91]: tips.groupby(['smoker', 'day']).apply(top, n=1, column='total_bill')
```
#### 分位数和桶分析
* 我们利用cut将其装入长度相等的桶中
* 要根据样本分位数得到大小相等的桶，使用qcut
```python
factor = pd.cut(frame.data1, 4, labels=False)
```

## 透视表和交叉表
### 透视表

```python
In [142]: tips.pivot_table(rows=['sex', 'smoker'])
Out[142]:
                   size       tip   tip_pct  total_bill
sex    smoker
Female No      2.592593  2.773519  0.156921   18.105185
       Yes     2.242424  2.931515  0.182150   17.977879
Male   No      2.711340  3.113402  0.160669   19.791237
       Yes     2.500000  3.051167  0.152771   22.284500
# 聚合tip_pct和size，并根据day进行分组，margins=True添加分项小计
In [144]: tips.pivot_table(['tip_pct', 'size'], rows=['sex', 'day'],
     ...:                  cols='smoker', margins=True)
Out[144]:
                 size                       tip_pct
smoker             No       Yes       All        No       Yes       All
sex    day
Female Fri   2.500000  2.000000  2.111111  0.165296  0.209129  0.199388
Sat   2.307692  2.200000  2.250000  0.147993  0.163817  0.156470
       Sun   3.071429  2.500000  2.944444  0.165710  0.237075  0.181569
       Thur  2.480000  2.428571  2.468750  0.155971  0.163073  0.157525
Male   Fri   2.000000  2.125000  2.100000  0.138005  0.144730  0.143385
       Sat   2.656250  2.629630  2.644068  0.162132  0.139067  0.151577
       Sun   2.883721  2.600000  2.810345  0.158291  0.173964  0.162344
       Thur  2.500000  2.300000  2.433333  0.165706  0.164417  0.165276
All          2.668874  2.408602  2.569672  0.159328  0.163196  0.160803
```
* 可以将任何对groupby 有效的函数传递给aggfunc
### 交叉表
* 交叉表是一种用于计算分组频率的特殊透视表

```python
In [150]: data
Out[150]:
   Sample  Gender    Handedness
0       1  Female  Right-handed
1       2    Male   Left-handed
2       3  Female  Right-handed
3       4    Male  Right-handed
4       5    Male   Left-handed
5       6    Male  Right-handed
6       7  Female  Right-handed
7       8  Female   Left-handed
8       9    Male  Right-handed
9      10  Female  Right-handed
In [151]: pd.crosstab(data.Gender, data.Handedness, margins=True)
Out[151]:
Handedness  Left-handed  Right-handed  All
Gender
Female                1             4    5
Male                  2             3    5
All                   3             7   10
```
