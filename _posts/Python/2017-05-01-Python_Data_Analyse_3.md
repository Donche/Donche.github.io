---
layout: post
title: Python数据分析笔记（3）：数据存储与规整化
category: 编程语言
catalog: true
lang: ch
tags: 
    - 2017
    - Python
    - 编程语言
---
# 数据加载、存储与文件格式
## 文本格式的数据

函数|说明
---|---
read_csv('filename.csv', sep=',')|从文件、url、文件型对象中加载带分隔符的数据。默认分隔符为逗号
read_table|从文件、url、文件型对象中加载带分隔符的数据。默认分隔符为制表符
read_fwf|读取定宽列格式数据
read_clipboard|读取剪贴板上的数据，可看作read_table 的剪贴板版

* 使用header设置用作列名的行号，用header=None 结合names=['a','b']指定用于结果的列名列表。
* 使用index_col='x' 来指定用作索引的列名。
* sep 里可为正则表达式。
* skiprows=[0,2,3] 可选择跳过的行数
* skip_footer 从文件末尾算起需要忽略的行数
* 用na_values=xx (xx为字典) 为各列指定不同的NA标记值
* nrows 表示需要读取的行数（从开头算起）
* chunksize 表示用于迭代的文件块的大小

### 逐块读取文本文件
设置chunksize 后read_csv 返回的对象可根据chunksize 对文件进行逐块迭代：
```python
chunker = pd.read_csv('ch06/ex6.csv', chunksize=1000)
tot = Series([])
for piece in chunker:
    tot = tot.add(piece['key'].value_counts(), fill_value=0)
tot = tot.order(ascending=False)
```

### 写出到文本格式
```python
data.to_csv('filename.csv',sep='|',na_rep='NULL',index=False, header=False, cols=['a', 'b', 'c'])
```
* Series 也有to_csv 和from_Series 方法

### JSON 数据
* 使用json.loads 可将JSON 字符串转换成Python 形式
* json.dumps(data) 将python 对象转换成JSON 形式

### XML 和HTML 相关处理
* 使用lxml库 读写HTML 和XML
* 使用```parse(urlopen('http://xxx.com/xxx')).getroot() ```可以得到特定类型的所有HTML标签
* 随后处理如下
```python
links = doc.findall('.//a')
links[x].get('href')
links[x].text_content()
```
* 对于XML 文件，使用objectify.parse 来解析
* root.INDICATOR 返回一个用于产生各个<INDICATOR>XML 元素的生成器

## 二进制数据格式
* 二进制存储最简单的方法是Python 内置的pickle 序列化。pandas 对象都有一个用于将数据pickle 形式保存的save 方法，而使用save 方法读取数据。（pickle 仅用于短期存储，因为其不稳定性）
### HDF5 格式
* HDF5（hierarchical data format）是一个流行的工业级C语言库，，带有许多语言的接口，能够存储多个数据集并支持元数据，还支持多种压缩器的即时压缩，还能更高效地存储重复模式数据。对于那些非常大的无法直接放入内存的数据集，HDF5就是不错的选择，因为它可以高效地分块读写。
* Python中的HDF5库有两个接口（即PyTables和h5py），它们各自采取了不同的问题解决方式。h5py提供了一种直接而高级的HDF5API访问接口，而PyTables则抽象了HDF5的许多细节以提供多种灵活的数据容器、表索引、查询功能以及对核外计算技术（out-of-core computation）的某些支持
### Microsoft Excel 文件
* 使用pd.ExcelFile('xxx.xls') 读取文件，使用xls_file.parse('sheet1') 解析某表格
* 写文件：df.to_excel('filename.xls', sheet_name='sheet1')

## 使用HTML 和Web API
* 使用requests 包访问API
```python
import requests
resp = requests.get(url)
import json
data = json.loads(resp.text)
```
如上，得到的data 里的results 里即为结果。

## 数据库
暂且跳过，以后细讲

# 数据规整化：清理、转换、合并、重塑

## 合并数据集
* pandas.merge 可根据一个或多个键将不同DataFrame中的行连接起来
* pandas.concat可以沿着一条轴将多个对象堆叠到一起

### 数据库风格的DataFrame 合并
* 使用merge 完成
* 最好显式指定on='key' 用来连接的列，否则默认使用重叠列的列名当作键
* 若列名不同可以分别指定：
```python
pd.merge(df3, df4, left_on='lkey', right_on = 'rkey')
```
* 默认使用how='inner' 连接，取交集。此外还有'right', 'left' 和'outer'
* 使用suffixes('\_left', '\_right') 追加同名列名末尾以区分
* sort 在连接键合并后排序，默认为开。在大数据集时关掉会有更好性能

### 索引上的合并
* left_index = True 说明索引应被用作连接键
* DataFrame 有一个join方法，可以方便的实现按索引合并，在连接键上做左连接。

### 轴向连接
* 使用pd.concat 将值与索引粘合在一起。当设定axis=1 时，返回DataFrame。通过join_axes 指定在其他轴上使用的索引。如：
```python
pd.concat([s1,s2], axis = 1, join = 'inner', join_axes = [['a', 'b', 'c']])
```
* 可用keys=['x','y','z'] 实现层次化索引。若沿着axis=1 对Series 合并，则keys 会成为DataFrame 的列头
* 使用ignore_index 忽略行索引

### 合并重叠数据
* 使用np.where 表达一种矢量化的if-else
* 使用combine_first 可实现相同的功能，并且会进行数据对齐。
```python
df.combine_first(df2)
```

## 重塑和轴向旋转

### 重塑层次化索引
* stack 将数据的列旋转为行。unstack 将数据的行旋转为列。

### 长格式旋转为宽格式
* pivot('x', 'y', 'value'): 前两个参数值分别用作行和列索引的列名。最后一个为数据列列名

## 数据转换

### 移除重复
* DataFrame 的duplicated() 返回一个布尔型Series 判断各行是否重复。使用drop_duplicates(['k1','k2']) 移除重复行，默认保留第一个值组合，使用take_last=True 保留最后一个

### 利用函数和映射进行数据转换
* 首先写一个需要转换的映射，然后用map处理Series。如：
```python
meat_to_animal = {
　'bacon': 'pig',
　'pulled pork': 'pig',
　'pastrami': 'cow',
　'corned beef': 'cow',
　'honey ham': 'pig',
　'nova lox': 'salmon'
}
data['animal'] = data['food'].map(str.lower).map(meat_to_animal)
#或用以下函数：
data['food'].map(lambda x: meat_to_animal[x.lower()])
```

### 替换值
* 值替换的一般形式：``` data.replace([-999, -1000], [np.nan, 0]) ```, 或 ``` data.replace({-999:np.nan, -1000:0}) ```

### 重名轴索引
* 轴标签也有一个map 方法
* 使用rename 创建数据集的转换版而不是修改原始数据，如``` data.rename(index=str.title, columns=str.upper) ```
* rename 还可结合字典对象对部分轴标签更新

### 离散化和面元划分
* 为了便于分析，连续数据会被离散化或拆分为面元(bin)
* 使用cut 函数划分为 “18-25”、“26-35”、“35-60”、“60以上” 几个面元：
```python
bins = [18, 25, 35, 60, 100]
cats = pd.cut(ages, bins, right=False， labels=group_names)#右开
```
* 使用levels 查看不同分类名称，value_counts(cats) 查看统计结果
* 还可传入数量而不是面元边界，这样的话会根据极值计算出等长面元
* 利用qcut 根据样本分位数对数据进行面元划分，可以得到大小基本相等的面元

### 检测和过滤异常值
* 举个例子，可通过以下代码限制值区间为[-3,3]
```python
data[np.abs(data) > 3] = np.sign(data) * 3
```

### 排列和随机采样
* 使用numpy.random.permutation 函数可实现列的排列工作。对轴的长度调用permutation 会产生一个新顺序的整数数组，随后可用ix 索引操作或take 函数使用数组。如：
```python
df = DataFrame(np.arange(5 * 4).reshape(5, 4))
sampler = np.random.permutation(5)
df.take(sampler)
```

### 计算指标/哑变量
* get_dummies，使用prefix 添加前缀：

```python
In [189]: df = DataFrame({'key': ['b', 'b', 'a', 'c', 'a', 'b'],
     ...:                 'data1': range(6)})

In [190]: pd.get_dummies(df['key'], prefix = 'key')
Out[190]:
   key_a  key_b  key_c
0  0  1  0
1  0  1  0
2  1  0  0
3  0  0  1
4  1  0  0
5  0  1  0
```
## 字符串操作
### 字符串对象方法
* 使用split + strip 使用：
```python
pieces = [x.strip() for x in val.split(',')]
```
* 使用字符串的join方法连接字符串：``` '::'.join(pieces) ```
* 使用find() 和 index() 定位字符串。若找不到index 引发异常，而find 返回-1
* 使用count() 返回出现次数，replace() 替换

### pandas 中矢量化的字符串函数
使用Series 的str属性访问方法

![shortCut](https://raw.githubusercontent.com/Donche/Donche.github.io/master/_posts/Python/SC_3_1.jpg)   
![shortCut](https://raw.githubusercontent.com/Donche/Donche.github.io/master/_posts/Python/SC_3_2.jpg)

---
*参考资料：《利用Python 进行数据分析》*
