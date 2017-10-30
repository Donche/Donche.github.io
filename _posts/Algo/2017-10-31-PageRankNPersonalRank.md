---
layout: post
title: PageRank 与基于图的推荐算法
category: 算法
catalog: true
mathjax: true
tags: 
    - 2017
    - 算法
    - 推荐系统
---

*为什么是PageRank和基于图的推荐算法呢？因为基于图的PersonalRank 算法[[1]](#1)就是从Topic-Sensitive PageRank 算法[[2]](#2)得来的。所以如果要彻底理解PersonalRank, 还是从大名鼎鼎的PageRank[[3]](#3)入手比较好*

# 1. PageRank
首先借用中文维基[[4]](#4)对PageRank的介绍：
>佩奇排名（PageRank），又称网页排名、谷歌左侧排名，是一种由搜索引擎根据网页之间相互的超链接计算的技术，而作为网页排名的要素之一，以Google公司创办人拉里·佩奇（Larry Page）之姓来命名。Google用它来体现网页的相关性和重要性，在搜索引擎优化操作中是经常被用来评估网页优化的成效因素之一。   

PageRank原理其实很简单：利用网页之间的链接关系来确定一个网页重要性，并且使用随机浏览的概念避免不存在外链的Dead End 中 Rank Sink 的问题，同时还可以此对网页进行个性化推荐[[3]](#3)。    

假设u 为一个页面，Fu 为u 指向的页面，Bu 为指向u 的页面，Nu 为u 指向的页面的数量，等于|Fu|。c 为归一化系数，则每个页面的rank 即为：   
$$R(u) = c \displaystyle\sum_{v\in B_u}\frac{R(v)}{N_v}$$    
但是，考虑一个简单的情况：当两个或者多个网页互相链接，并且没有其他向外的链接的时候，在迭代过程中，这几个网页的rank 值会不断增加趋近于1，其他网页则趋近于0，引发名为 Rank Sink。为了解决这个问题，我们假设用户并不只是一直在点击网页内的超链接，而会有一定概率在所有网页范围中随机浏览一些网页（random surfer）。于是在每次计算的时候考虑随机浏览网页概率的向量E 的公式如下：      
$$R'(u) = c \displaystyle\sum_{v\in B_u}\frac{R'(v)}{N_v} + cE(u)$$     
用矩阵来表示即为：   
$$R'(u) = c(AR' + E)$$   
*R’ 的一范数为1*    
所以可以得到：    
$$R'(u) = c(A + E * 1) * R'$$   
即， R’为(A + E*1) 的一个特征向量。如果使用另一种记号[[2]](#2)，定义随机浏览的概率为alpha，公式还可写为：    
$$Rank = (1-\alpha)M * Rank + \alpha p$$   
E（或p）的引入不仅解决了rank sink的问题，同时还可以通过E 的取值为用户提供个性化推荐，一举两得。    


# 2. Topic-Sensitive PageRank
但是以上算法忽视了一个问题：用户的对于网页重要性的评判是不一样的。尽管可以使用不同的E 来个性化搜索结果，但是当用户数量大幅增加的时候，维护规模如此庞大的矩阵是很困难的。所以便出现了Topic-Sensitive PageRank[[2]](#2), 采用主题的方式，维护若干个主题的向量，然后关联用户与主体之间的相关度，以此为用户推荐搜索结果。Topic-Sensitive PageRank 的计算公式如下：    
$$Rank = (1-\alpha)M * Rank + \alpha v$$    
其中v 为每个主题各自维护，记录所有页面与主题的关系。假如有A, B, C, D 这四个主题，每个主题各自维护一个v 向量，有a, b, c, d, e 这五个网站，其中b, d 与B 主题相关，则$$v_b$$ 向量为 {0，0.5， 0， 0.5， 0 }。   
得到每个主题的rank 值之后，便可以通过各种方式得到用户喜好，向其推荐对应topic 的链接即可。    

# 3. PersonalRank
因为《推荐系统实践》中关于PersonalRank 算法[[1]](#1)并没有介绍太多实际实施的内容，书中介绍的迭代算法对于movielens这样的比较大的数据集基本是无力求解的，但是还介绍了一个斯坦福的博士论文，打开一看168页，心想有时间再看吧。查了一下，python中已经有了稀疏矩阵相关的方法，于是造轮子的事情就先搁置吧...   
## 3.1 PersonalRank 模型：
引用《推荐系统实践》中对PersonalRank 的描述如下：
>假设要给用户u进行个性化推荐，可以从用户u对应的节点vu开始在用户物品二分图上进行随
机游走。游走到任何一个节点时，首先按照概率α决定是继续游走，还是停止这次游走并从vu节
点开始重新游走。如果决定继续游走，那么就从当前节点指向的节点中按照均匀分布随机选择一
个节点作为游走下次经过的节点。这样，经过很多次随机游走后，每个物品节点被访问到的概率
会收敛到一个数。最终的推荐列表中物品的权重就是物品节点的访问概率。    

原理上与PageRank是很相似的，区别在于PageRank 中的链接是有向的，而PersonalRank 人于物品之间的连接是无向的，或者说是双向的。迭代公式如下：
$$ f(n) = 
\begin{cases}
    \alpha \displaystyle\sum_{v'\in (v)}\frac{PR(v')}{|out(v')|}       & \quad \text{if }(v \neq v_u)\\
    (1- \alpha) + \alpha \displaystyle\sum_{v'\in (v)}\frac{PR(v')}{|out(v')|}   & \quad \text{if }(v = v_u)  
\end{cases}
$$   


写为矩阵形式为：     
  
$$Rank = (1-\alpha)M^T Rank + \alpha r_0$$   
其中$$r_0$$可以看作Topic-Sensitive PageRank中的v向量，表示每次随机游走有α 的概率回到起始节点。最终我们得到了每个物品和人的rank 如下：    
$$Rank = \alpha (I- (1-\alpha)M^T)^{-1}R_0 $$     
或者写成线性方程组的形式：    
$$(I- (1-\alpha)M^T)Rank =\alpha R_0 $$    
因此，为了求出Rank，只需要解上述方程即可，比迭代算法要快很多了。

## 3.2 求解

事实上大型稀疏矩阵求逆还是一个挺复杂的问题，尤其是在系数矩阵求逆后一般都不是稀疏矩阵的情况下。所以解以上方程还可以考虑使用稀疏矩阵求解线性方程组的方法。这两种方法都集成在了scipy 中，所以实现起来还是比较方便的。然而因为方程组求解很慢，面对大量的用户求解会很费时间，所以我使用了[movielens 100k](https://grouplens.org/datasets/movielens/)的数据集，

### 3.2.1 生成稀疏矩阵
在scipy 中生成稀疏矩阵一个方法是先创建稀疏矩阵然后填充元素。因为每行的值相加都为1，而且每个节点到周围节点的概率都相等，所以每个值概率为 1/Out(u)。代码如下：

```python
def makeMatrixA(train, item_users, itemNum,  alpha):
    userNum = len(train)
    M = lil_matrix((userNum + itemNum,userNum +itemNum))
    for item, users in item_users.items():
        val = 1. / len(users)
        for user in users:
            M[userNum + item - 1, user - 1] = val
    for user, items in train.items():
        val = 1. / len(items.keys())
        for item in items.keys():
            M[user - 1, userNum + item - 1] = val
    #求解线性方程组的系数矩阵
    A = lil_matrix(np.eye(userNum + itemNum)- (1-alpha) * M.T)
    #逆矩阵
    A = inv(M)
    return A

```


### 3.2.1 求解线性方程组
scipy中有用于求解稀疏矩阵线性方程组的广义最小残量(gmres)方法[[5]](#5)。需要注意的是b只能为一个向量且不能为稀疏矩阵的格式。实现代码如下：
```python
def solveRankLinalg(trainUser, A, user, userNum):
    b = np.zeros((A.shape[0],1))
    b[user - 1] = 1
    r = gmres(A, b, tol=1e-08, maxiter=1)[0][userNum:]
    rank = {}
    for i in range(len(r)):
        if i+1 not in trainUser:
            rank[i + 1] = r[i]
    return sorted(rank.items(), key = lambda x: x[1], reverse = True)[0:10]
```

### 3.2.2 求解逆矩阵
稀疏矩阵求解逆矩阵可以使用scipy.sparse.linalg.inv() 方法，但是经过和matlab的对比发现准确度并不是很高，而且也没有可以调整准确度的参数，所以最终结果并没有求解线性方程组的好，但是速度却快了好几倍。因为逆矩阵得到之后，每个用户的rank值即为矩阵的对应列乘个系数，所以很方便。代码如下：
```python
def invRank(invM, train, userNum, itemNum, items_pool):
    rank = dict()
    for user in range(len(train)):
        rankU = invM[:,user].A
        rank_tmp = dict()
        rankUIndex = sorted(range(len(rankU)-userNum), key=lambda k: rankU[k+userNum], reverse= True)
        for item in rankUIndex:
            if item+1 not in train[user+1]:
                rank_tmp[item+1] = rankU[item][0]
        rank[user + 1] = dict(sorted(rank_tmp.items(), key = lambda x: x[1], reverse = True)[0:10])
    return rank
```
代码中的invM 每行每列前userNum 行/列 为用户节点，之后为物品节点。需要特别注意数据集中物品和用户计数都是从1开始，而python 矩阵是从0开始，所以第N 列其实代表的是第N+1 个用户或者第N-userNum+1个物品。



## 3.3 结果
因为求逆矩阵的方法准确率不是很高（或许是因为求解逆矩阵误差太大？），所以我用求解线性方程组的方法得到了不同α 下的结果：

alpha   | recall    | precision | coverage  | popularity    | hit number
---     | ---       | ---       | ---       | ---           |  ---
0.1     | 9.32%     | 29.63%    | 2.75%     | 5.7243        | 2794
0.3     | 10.45%    | 33.24%    | 4.40%     | 5.6751        | 3135
0.5     | 11.06%    | 35.19%    | 4.95%     | 5.6412        | 3318
0.7     | 11.35%    | 36.09%    | 5.50%     | 5.6249        | 3403
0.9     | 11.51%    | 36.61%    | 5.69%     | 5.6186        | 3452
0.95    | 11.52%    | 36.63%    | 5.69%     | 5.6179        | 3454
1       | 2.70%     | 8.58%     | 2.02%     | 4.6462        | 809


可以看出α 取0.9的时候各个指标达到了最好效果。但是如果将结果与协同过滤算法相比，虽然准确率和召回率达到了相似的水平，但是覆盖率却很低。根据《推荐系统实践》书中得到的数据，看来personalRank 的整体覆盖率确实不是很高。


*参考资料*
1. <span id="1"></span>《推荐系统实践》
2. <span id="2"></span>Taher H. Haveliwala,Topic-Sensitive PageRank, 2002, 
3. <span id="3"></span>Page, Lawrence and Brin, Sergey and Motwani, Rajeev and Winograd, Terry (1999) The PageRank Citation Ranking: Bringing Order to the Web. Technical Report. Stanford InfoLab.
4. <span id="4"></span>[佩奇排名-维基百科](https://zh.wikipedia.org/zh-hans/%E4%BD%A9%E5%A5%87%E6%8E%92%E5%90%8D?oldformat=true)
5. <span id="5"></span>[广义最小残量方法-维基百科](https://zh.wikipedia.org/wiki/%E5%B9%BF%E4%B9%89%E6%9C%80%E5%B0%8F%E6%AE%8B%E9%87%8F%E6%96%B9%E6%B3%95?oldformat=true)
6. [浅析PageRank算法--博客] (http://blog.codinglabs.org/articles/intro-to-pagerank.html)
7. [PersonalRank：一种基于图的推荐算法---博客园](http://www.cnblogs.com/zhangchaoyang/articles/5470763.html)