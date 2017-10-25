---
layout: post
title: 基于邻域的推荐算法
category: 算法
catalog: true
tags: 
    - 2017
    - 算法
    - 推荐系统
---
<script type="text/javascript" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default">
</script>
# 基于邻域的算法
基于邻域的算法分为两大类，一类是基于用户的协同过滤算法，另一类是基于物品的协同过滤算法。这两类算法都已经很成熟，所以做个笔记以备忘。数据集使用MovieLens提供的1M 数据集，可从[这个网站上](https://grouplens.org/datasets/movielens/) 下载。读取数据集并按7：3将每个用户看过的电影分为训练集和测试集。

*本博客基于《推荐系统实践》(1)*
# 基于用户的协同过滤算法
这个算法主要分为两部分：
1. 计算用户相似度
2. 根据用户的相似度给用户生成推荐列表


## 计算用户相似度
假设有用户u 和 v， 他们之间的相似度为：

$$\omega_{uv}=\frac{|N(u)\bigcap N(v)|}{|N(u)\bigcup N(v)|}$$
或者使用余弦相似度：
$$\omega_{uv}=\frac{|N(u)\bigcap N(v)|}{\sqrt{|N(u)||(v)|}}$$

根据这两个公式，我们需要分别计算分子分母。假如计算方式为两两分别计算的话，当用户量变大，大部分用户之间都没有共同喜欢的物品，计算效率会非常低。所以我们先统计出一个 物品-用户 的倒排表，然后遍历这个倒排表统计出每两个用户之间感兴趣物品的集合，具体代码如下：
```python
def UserSimilarity(train):
    #建立倒排表
    item_users = dict()
    for u, items in train.items():
        for i in items.keys():
            if i not in item_users:
                item_users[i] = set()
            item_users[i].add(u)
    #计算用户矩阵
    C = dict()
    N = dict()
    for i, users in item_users.items():
        for u in users:
            N[u] = N.get(u,0) + 1
            for v in users:
                if u == v:
                    continue
                if u not in C:
                    C[u] = dict()
                #C[u][v] = C[u].get(v,0) + 1
                C[u][v] = C[u].get(v,0) + 1 / math.log(1 + len(users))
    #计算用户相似度
    W = dict()
    for u, related_users in C.items():
        if u not in W:
            W[u] = dict()
        for v, cuv in related_users.items():
            W[u][v] = cuv / math.sqrt(N[u] * N[v])
    return W

```
其实在以上代码已经顺便统计出了N(u)，所以用户的相似度便得到了。

所以，我们进入下一步。   
并没有。   
现在假设有一些用户，都喜欢看电影（那当然）。他们都看过 《星球大战》、《这个杀手不太冷》、《肖申克的救赎》等等等等几十部热门电影......那么他们的兴趣很相近吗？在更多的可能性下，其中一些迷恋塔可夫斯基、一些是诺兰脑残粉、一些沉迷希区柯克、一些是漫威死宅，还有一些是文艺小清新。万一给岩井俊二的迷妹推荐到希区柯克的电影并且对该用户造成一百八十平米的心理阴影，那么这个推荐系统就很失败了。   
所以如何避免这些热门电影过度影响用户兴趣相似度的计算呢？so easy，减个权重就行。公式如下：
$$\omega_{u,v} = \frac{\sum_{i\in N(u)\bigcap N(v)} \frac{1}{log1 + |N(i)|}}{\sqrt{|N(u)||N(v)|}}$$

## 物品推荐
得到了与目标用户u兴趣最相近的K个用户后，就可以推荐这些用户感兴趣的而目标用户u并没有听说过的物品。其中每个物品i 对于用户u 的权重可以用以下公式计算：
$$p(u,i)=\displaystyle\sum_{v\in S(u,K)\bigcap N(i)} \omega_{u,v}r_{v,i} $$

如果使用单一的隐反馈数据的话，所有的r 都等于1。如果还想使用评分数据的话，可以用评分来当作权数（还没有试过）。得到的结果通过评分排序后，选取前十个返回（Top10）
```python
def Recommend(user, train, W, K):
    rank = dict()
    interacted_items = train[user]
    for v, wuv in sorted(W[user].items(), key=itemgetter(1), reverse=True)[0:K]:
        for i, rvi in train[v].items():
            if i in interacted_items:
                #we should filter items user interacted before
                continue
            if i not in rank:
                rank[i] = 0
            rank[i] += wuv * rvi
    return sorted(rank.items(), key = itemgetter(1), reverse = True)[0:10]
```

## 指标测试
正如上节所说，衡量一个推荐算法最重要的有以下几个参数：
* 准确率和召回率
* 流行度
* 覆盖率
测试时可以用交叉验证，更准确,也更费时间。我直接用测试集测试。具体代码如下，同时返回了所有的指标：
```python
def PrecisionRecall(test, train, W, K):
    hit = 0
    n_recall = 0
    n_precision = 0
    recommend_items = set()
    all_items = set()
    item_popularity = dict()
    #统计所有物品以及出现的次数
    for user, items in train.items():
        for item in items.keys():
            if item not in item_popularity:
                all_items.add(item)
                item_popularity[item] = 0
            item_popularity[item] += 1
    popularity = 0
    n = 0
    for user, items in test.items():
        rank = Recommend(user, train, W, K)
        hitkeys = set(dict(rank).keys()).intersection(items)
        hit += len(hitkeys)
        n_recall += len(items)
        n_precision += len(rank)
        for item, pui in rank:
            recommend_items.add(item)
            popularity += math.log(1 + item_popularity[item])
            n += 1
    coverage = len(recommend_items) / (len(all_items) * 1.0)
    popularity /= n * 1.0
    return [hit / (1.0 * n_recall), hit / (1.0 * n_precision), hit, n_recall, n_precision, coverage, popularity]
```
通过不同的用户数K， 得到以下结果：

K   | recall| precision | coverage  | popularity    | hit number
--- | ---   | ---       | ---       | ---           |  ---
5   | 5.72% | 28.40%    | 55.03%    | 6.5460        | 17154
10  | 6.70% | 33.28%    | 44.76%    | 6.7169        | 20102
20  | 7.54% | 37.48%    | 34.42%    | 6.8582        | 22637
40  | 8.05% | 40.00%    | 26.16%    | 6.9771        | 24160
80  | 8.46% | 42.02%    | 19.89%    | 7.0811        | 25378


可以看出，推荐系统在K = 80 的情况下得到了最高的准确率和召回率，但是覆盖率和流行度并不理想，尤其是覆盖率。

# 基于物品的协同过滤算法

一切都与基于用户的协同过滤算法那么相似：
1. 计算物品的相似度
2. 根据物品的相似度和用户的历史行为给用户生成推荐列表

## 计算物品的相似度
依然是那么相似：
假设有物品u 和 v， 他们之间的相似度为：
$$\omega_{uv}=\frac{|N(u)\bigcap N(v)|}{|N(u)\bigcup N(v)|}$$
或者使用余弦相似度：
$$\omega_{uv}=\frac{|N(u)\bigcap N(v)|}{\sqrt{|N(u)||(v)|}}$$
同上，我们先统计出一个 用户-物品 的倒排表，然后遍历这个倒排表统计出每两个物品之间相似的集合。因为我们的数据集本身就是 用户-物品 排列的，所以可以省掉一部分处理。具体代码如下：
```python
def ItemSimilarity(train):
    C = dict()
    N = dict()
    for u, items in train.items():
        for i in items.keys():
            N[i] = N.get(i,0) + 1
            for j in items.keys():
                if i == j:
                    continue
                if i not in C:
                    C[i] = dict()
                C[i][j] = C[i].get(j,0) + 1 / math.log(1 + len(items) * 1.0)
    #计算相似矩阵
    W = dict()
    for i,related_items in C.items():
        W[i] = dict()
        for j, cij in related_items.items():
            W[i][j] = cij / math.sqrt(N[i] * N[j])
    return W
```
同样，我们依然需要对超级活跃的用户进行惩罚。几个阅片无数的用户很容易使得很多不相关的冷门物品出现相似性。惩罚方法与之前基于用户的协同过滤算法一样。

## 物品推荐
用户u对物品i的兴趣由以下公式计算(跟上一个其实是一样的)：
$$p(u,i)=\displaystyle\sum_{v\in S(i,K)\bigcap N(u)} \omega_{i,v}r_{u,v} $$

基于物品的协同推荐的一个好处是，可以提供推荐理由：根据您喜欢的XXXX推荐。

>Karypis 在研究中发现如果将ItemCF的相似度矩阵按最大值归一化，可以提高推荐的准确率。其研究表明，如果已经得到了物品相似度矩阵w，那么可以用如下公式得到归一化之后的相似度
矩阵w'：  
$$\omega_{ij}' = \frac{\omega_ij}{max_j \omega_{ij}}$$


于是我做个了小测试，结果如下：

类别  | recall    | precision | coverage  | popularity    | hit number
---     | ---   | ---   | ---   | --- |  ---
未归一化 | 5.72% | 28.40% | 55.03% | 6.5460 | 17154
归一化 | 0.05% | 0.25% | 19.18% | 2.2626 | 153


感觉这样的结果应该是我代码写错了？...找了好久还没有找到bug，找到了再更新吧   


## 指标测试
用同样的方法和参数跑了~~好多好多遍~~算法，得到以下结果:     


K   | recall| precision | coverage  | popularity    | hit number
--- | ---   | ---       | ---       | ---           |  ---
5   | 7.70% | 38.24%    | 21.34%    | 7.0742        | 23097
10  | 7.86% | 39.03%    | 18.33%    | 7.1771        | 23573
20  | 7.85% | 39.02%    | 16.15%    | 7.2462        | 23570
40  | 7.71% | 38.32%    | 14.75%    | 7.2805        | 23148
40  | 7.55% | 37.53%    | 13.39%    | 7.2929        | 22666


由表看出，K = 5 的时候准确率和召回率达到了峰值。而覆盖率和流行度随K 的增大单调递减。

# 对比分析

## 数据量上的区别
例如在新闻网站等内容更新非常快的网站，基于物品的协同过滤算法造成很大的计算量，并且占用大量空间存储物品关系矩阵，性能并不会好。同时每天需要更新如此大的矩阵也是不现实的。这时我们可以根据用户之间的相似度推荐物品。    
再例如在电影推荐系统中，电影更新很慢，数量可能远少于用户数量。基于物品推荐就很合理。其次例如购物网站等也是同理。

## 时效性的区别
又拿新闻网站举例子。作为时效性很强的应用，过期的新闻很少有人看，同时网站每天会产生大量的新的信息，仅仅是维持这个矩阵就已经很难实现。相反用户的增长则没有那么多，更新用户的相似度更加可行。   
再例如在购物网站中，用户并不需要在大量新上市的产品中挑选，而且用户的购买兴趣很少在短时间内有巨大变化，这样使用基于物品的推荐便成为了首选


# 利用multiprocessing 加速
首先转载其他博客（2）的介绍：
>python中的多线程其实并不是真正的多线程，如果想要充分地使用多核CPU的资源，在python中大部分情况需要使用多进程。Python提供了非常好用的多进程包multiprocessing，只需要定义一个函数，Python会完成其他所有事情。借助这个包，可以轻松完成从单进程到并发执行的转换。multiprocessing支持子进程、通信和共享数据、执行不同形式的同步，提供了Process、Queue、Pipe、Lock等组件。     
>在利用Python进行系统管理的时候，特别是同时操作多个文件目录，或者远程控制多台主机，并行操作可以节约大量的时间。当被操作对象数目不大时，可以直接利用multiprocessing中的Process动态成生多个进程，十几个还好，但如果是上百个，上千个目标，手动的去限制进程数量却又太过繁琐，此时可以发挥进程池的功效。
>Pool可以提供指定数量的进程，供用户调用，当有新的请求提交到pool中时，如果池还没有满，那么就会创建一个新的进程用来执行该请求；但如果池中的进程数已经达到规定最大值，那么该请求就会等待，直到池中有进程结束，才会创建新的进程来它。

multiprocessing.Pool 应该是在jupyter notebook上用不了，我的实验以失败告终。   
关于Pool 的介绍改天专门讲，这里只说具体用法。    
multiprocessing 的Pool 可以自动生成多进程的python 应用，方便易用，只需要改动少数几个地方即可。首先当然是 ``` import multiprocessing ```    
因为Pool 必须在 __main__ 里运行，所以必须加上 ```if __name__ == "__main__"``` 这一行。然后将需要调用的函数和参数传递给Pool.map()即可，代码如下：
```python
pool = multiprocessing.Pool(2)
rank_res = pool.map(Recommend,inp)
pool.close()
pool.join()
```
上述代码创建了两个进程，分别将inp 内的每个值传递给Recommend 函数。随后关闭进程池并等待所有进程完毕。      
这里需要稍微注意修改以下Recommend 函数的输入以及输出。由于是在循环外调用推荐函数，所有的结果都会存储在rank_res 里，为list 形式，需要手动转为dict。


效果当然是很显著的。使用ItemCF，当N = 5时，不使用多进程所用全部计算时间为1205s，使用4个进程同时计算，所需时间降低到了552s


*公式用 MathJax 引擎生成*

*参考资料*
1. 《推荐系统实践》 项亮
2. http://www.cnblogs.com/kaituorensheng/p/4445418.html   
3. http://www.cnblogs.com/whatisfantasy/p/6440585.html


# 附录
n_recall 300086 n_precision 60400      
ItemCF    
7：3    
5：recall 0.05% precision 0.25% hit 153 coverage 19.18% popularity 2.2626 归一化   
5 recall: 5.72%  precision: 28.40  hit: 17154  coverage: 55.03 popularity: 6.5460   
10 recall: 6.70	precision: 33.28 hit: 20102	coverage: 44.76	popularity: 6.7169   
20 recall: 7.54	precision: 37.48 hit: 22637	coverage: 34.42	popularity: 6.8582   
40 recall: 8.05	precision: 40.00 hit: 24160	coverage: 26.16	popularity: 6.9771   
80 recall: 8.46	precision: 42.02 hit: 25378	coverage: 19.89	popularity: 7.0811   
3:1   
5: recall 7.70% precision 38.24% hit 23097 coverage 21.34% popularity 7.0742    
10：recall 7.86% precision 39.03% hit 23573 coverage 18.33% popularity 7.1771   
20: recall 7.85% precision 39.02% hit 23570  coverage 16.15% popularity 7.2462   
40: recall 7.71% precision 38.32% hit 23148 coverage 14.75% popularity 7.2805   
80: recall 7.55% precision 37.53% hit 22666 coverage 13.39% popularity 7.2929   

UserCF   
5:  recall 6.14% precision 25.20% hit 15224 coverage 54.33% popularity 6.6178   
10: recall 7.32% precision 30.04% hit 18145 coverage 43.21% popularity 6.7914   
20: recall 8.26% precision 33.91% hit 20482 coverage 34.31% popularity 6.9309   
40: recall 8.96% precision 36.79% hit 22223 coverage 26.33% popularity 7.0461   
80: recall 9.27% precision 38.06% hit 22993 coverage 20.66% popularity 7.1460   
160:recall 9.32% precision 38.26% hit 23112 coverage 15.44% popularity 7.2343   
