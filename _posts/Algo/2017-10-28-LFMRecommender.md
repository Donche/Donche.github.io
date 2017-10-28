---
layout: post
title: 隐语义模型（LFM）推荐算法
category: 算法
catalog: true
mathjax: true
tags: 
    - 2017
    - 算法
    - 推荐系统
---

*当然，除了之前说的基于邻域的推荐算法，还有很多其他的算法，比如隐语义模型。因为《推荐系统实践》[[1]](#1)里面介绍过，但是公式和代码都有一些错误，而且一些代码缺少解释，~~同时因为我花了大量时间debug调参数调到怀疑人生的地步~~，所以写个博客记录一下*

# 1. 隐语义模型
>隐语义模型是最近几年推荐系统领域最为热门的研究话题，它的核心思想是通过隐含特征
(latent factor)联系用户兴趣和物品。[[1]](#1)

隐语义模型的实质就是对用户和物品进行分类。根据一个用户兴趣的分类对他推荐该分类下的物品。   
那么问题来了，如何对用户的兴趣进行分类？如何对物品进行分类？如何确定物品在所在类别中的权重？    
如果请专家分类鉴别确定权重，肯定会造成...（省略253字的缺点分析）。总之，我们是不能人为对这么大的数据集分类的，免费劳动力也不行。   

那怎么办？   
大佬David Wheeler 曾说过，没有什么计算机问题是加个中间层解决不了的，~~如果一个不行，那就加两个。~~[[2]](#2)    
那就从用户和物品之间加一层系数好了。从数据出发，使用算法自动得到物品和用户的分类权数。不仅准确度更高，可以得到可靠的权重，还减少了标记物品所需要的人力。

# 2. LFM 的实现
LFM 通过以下模型得到用户u 对物品i 的兴趣：
$$Preference(u,i) = r_{ui} = p_u^Tq_i = \sum_{k=1}^{F}p_{u,k}q_{i,k}$$   
*书里将k错写为f*     
其中，k为隐类，p_uk为用户u 与隐类k之间的关系，q_ik为物品i与隐类k之间的关系。两者相乘即为该用户与该物品之间的权重。    

## 2.1 选取正负样本
为了取得p, q 这两个参数，我们需要对每个用户取正负样本若干，分析用户喜好。对于电影评分问题，取得正负样本都比较方便。但是对于隐形反馈数据集，即只有正样本的数据集来说，负样本的选择就很重要了。通过2011年的KDD Cup的Yahoo! Music推荐系统比赛，我们发现负样本的选取需要遵循以下原则[[1]](#1)：
1. 保证每个用户正负样本的平衡
2. 对每个用户采样负样本时，要选取那些很热门，而用户却没有行为的物品。   
仔细思考一下，第二点在逻辑上是很合理的。如果在某个大片火热上映，演艺圈大佬主动站台，亲友热情推荐之下，我依然没有选择这个电影，那估计我就是真不喜欢它了。这样未被选择的热门物品在一般情况下比未选择的冷门物品更能体现一个人的兴趣偏好。书中提供的负采样代码如下：
```python
def RandomSelectNegativeSample(self, items):
    ret = dict()
    for i in items.keys():
        ret[i] = 1
    n = 0
    for i in range(0, len(items) * 3):
        item = items_pool[random.randint(0, len(items_pool) - 1)]
        if item in ret:
            continue
        ret[item] = 0
        n + = 1
        if n > len(items):
            break
    return ret
```
需要注意的是，代码中的items_pool 为一个存储着所有items 的list，因为有重复，所以每个物品被选中的概率和它出现的频率成正比。这就完成了选取热门物品的部分。   
但实际使用中还有另外一种方法，就是每次都选取最热门的几个物品[[4]](#4)，代码如下:
```python
def RandSelectNegativeSamples(items, items_pool):
    ret = dict()
    for i in items.keys():
        ret[i] = 1
    posiSize = len(ret)
    n = 0
    negativeSamples = sorted(items_pool.items(), key = itemgetter(1), reverse = True)
    for item, v in negativeSamples:
        if n > posiSize:
            break
        if item not in ret:
            ret[item] = 0
        n += 1
    return ret
```
代码中的items_pool 为一个dict, 存储着所有物品和出现次数。通过对它排序得到最热门物品，随后从其中选出与正样本数量一致的负样本数量并将值置为1。但这种方法效果并没有书中给出的方法好，所以并没有采用。       
在我实践的时候还发现，在对每个物品均匀地区负样本效果更好，这似乎与书中的结论有矛盾。这一点还有待考察...


## 2.2 优化目标函数
随后，通过优化目标函数来获得准确的p，q：
$$ C = \displaystyle\sum_{(u,i)\in K}(r_{ui} - \hat{r_{ui}})^2  = \displaystyle\sum_{(u,i)\in K}(r_{u,i} - \displaystyle\sum_{k=1}^{K}p_{u,k}q_{i,k})^2 + \lambda ||p_u||^2 + \lambda ||q_i||^2 $$    
其中后两项为正则项，防止过拟合。如果有一些机器学习基础的话，这个方程形式是很熟悉的了。通过梯度下降法来求解是最常见的。对目标函数每项分别求偏导，得到以下函数：    
$$\frac{\partial C}{\partial p_{uk}} = 2(r_{ui} - \displaystyle\sum_{k=1}^Kp_{uk}q_{ik})(-q_{ik}) + 2\lambda p_{uk} $$   
$$\frac{\partial C}{\partial q_{ik}} = 2(r_{ui} - \displaystyle\sum_{k=1}^Kp_{uk}q_{ik})(-p_{uk}) + 2\lambda q_{ik} $$   
*书里公式有错误，但代码逻辑是正确的*      
使用梯度下降法，得到以下递推公式：
$$p_{uk} = p_{uk} - \alpha \frac{\partial C}{\partial p_{uk}}$$   
$$q_{ik} = q_{ik} - \alpha \frac{\partial C}{\partial q_{ik}}$$   
其中，α 是学习速率，需要实验得出。综上，书中给出的代码如下（可以使用向量化以加速，改进后的代码在[这里](#NegaSample)）：
```python
def LatentFactorModel(user_items, F, N, alpha, flambda):
    users_pool = []
    items_pool = []
    for user, items in user_items.items():
        users_pool.append(user)
        for item in items.keys():
            items_pool.append(item)
    [P, Q] = InitModel(users_pool, items_pool, F)
    for step in range(0,N):
        for user, items in user_items.items():
            samples = RandSelectNegativeSamples(items, items_pool)
            for item, rui in samples.items():
                eui = rui - Predict(P[user], Q[item])
                for f in range(0, F):
                    P[user][f] += alpha * (eui * Q[item][f] - flambda * P[user][f])
                    Q[item][f] += alpha * (eui * P[user][f] - flambda * Q[item][f])
        alpha *= 0.9
    return [P, Q]
``` 
以上代码中出现了InitModel 这个函数，但书中并没有解释。具体实现时，可以将p, q 的每个值初始化为1------这样当然是不行的。如果初始化为同一个值的话，此后每个用户和物品所属的隐类权重都会是一样的。所以，具体实现时，可以初始化为0到1之间的随机数：```random.random()```     
同样没有提到的还有Predict(P[user], Q[item]) 这个~~我完全没有留意结果被坑的~~函数。因为需要输出的rank 值为0-1之间，所以在将所有系数相乘求和之后，还需要用sigmoid 函数处理一下：
```python
def Predict(Puser, Qitem):
    rank = 0
    for f,puf in Puser.items():
        rank += puf * Qitem[f]
    rank = 1.0/(1+math.exp(-rank)) 
    return rank
```
## 2.3 物品推荐
向用户推荐物品时，将p,q 的每隔隐类的系数相乘然后相加，即可得到最终权数，代码如下：
```python
def Recommend(user, P, Q):
    rank = dict()
    for f, puf in P[user].items():
        for item in Q.keys():
            if item not in rank:
                rank[item] = 0
            rank[item] += puf * Q[item][f]
    return sorted(rank.items(), key = lambda x: x[1], reverse = True)[0:10]
```

# 3. 可能遇到的问题
## 3.1 P，Q 中出现Nan
我首先使用书中的参数，固定F=100，N = 18，alpha=0.02、lambda=0.01，ratio = 1，结果很惨淡，召回率在百分之一左右，P, Q 矩阵内很多数都为Nan。于是找了大半天bug，最终在小数据集上调试发现原来p，q 发散了...这个悲惨的故事告诉我们调参还是得自己来，~~伸手得来的都是不可靠的。~~    
同时，得到结果是Nan 还可能是因为数据量小+冷启动问题导致的[[3]](#3)。检查了一遍之后发现我的数据里并没有这个情况。
## 3.2 负样本选取
因为每次迭代循环都会取不同的负样本，使误差函数一直维持在很高的水平，但是效果确实会随迭代次数的增加而改善。   
我还参考了另外一种负样本选取方法[[4]](#4)，将负样本的选择放在循环之外，每次用确定的负样本进行迭代。结果发现效果差了很多倍，所以还是不建议这种方法。具体代码如下：
```python
def LatentFactorModel(user_items, F, N, alpha, flambda):
    users_pool = []
    items_pool = dict()
    samples = dict()
    for user, items in user_items.items():
        users_pool.append(user)
        for item in items.keys():
            if item not in items_pool:
                items_pool[item] = 0
            items_pool[item] += 1
    for user in users_pool:
        samples[user] = RandSelectNegativeSamples(items, items_pool)
    [P, Q] = InitModel(users_pool, items_pool, F)
    for step in range(0,N):
        for user, items in user_items.items():
            for item, rui in samples[user].items():
                eui = rui - Predict(P[user], Q[item])
                for f in range(0, F):
                    P[user][f] += alpha * (eui * Q[item][f] - flambda * P[user][f])
                    Q[item][f] += alpha * (eui * P[user][f] - flambda * Q[item][f])
        alpha *= 0.9
    return [P, Q]
```

## 3.3 <span id="NegaSample"></span>使用向量化加速
因为代码中包含大量数组的相乘和求和，使用dict + 循环求和的方式很显然效率很低，于是可以考虑使用numpy 来加速。将P，Q 写为numpy ndarray 的形式，可以大大加快程序运行速度。实验发现仅仅是将P[user] 与Q[item] 换为一维数组就已经可以可以将计算时间缩小7倍左右。具体实现时需要将P，Q的每一项初始化为 ```np.random.rand(1,F)```，其中F为隐类的数量。P，Q的迭代代码如下：
```python
def LatentFactorModel(user_items, F, N, alpha, flambda, falpha):
    users_pool = []
    items_pool = dict()
    #samples = dict()
    for user, items in user_items.items():
        users_pool.append(user)
        for item in items.keys():
            if item not in items_pool:
                items_pool[item] = 0
            items_pool[item] += 1
    [P, Q] = InitModel(users_pool, items_pool, F)
    for step in range(0,N):
        err_sum = 0
        for user, items in user_items.items():
            samples = RandSelectNegativeSamplesNouse(items, list(items_pool.keys()))
            for item, rui in samples.items():
                eui = rui - Predict(P[user], Q[item])
                err_sum += eui
                P[user] += alpha * (eui * Q[item] - flambda * P[user])
                Q[item] += alpha * (eui * P[user] - flambda * Q[item])
        alpha *= falpha
        print(err_sum)
    return [P, Q]
```
需要注意的是，因为使用的是stochastic gradient descent, 所以并不能再将上一层的循环向量化，否则很容易发散溢出(一个上午换来的教训...)。


# 4. 结果分析
在调参过程中发现alpha, 迭代次数 和 正负样本的比例都很重要，结果如下：   
F = 100, lambda = 0.01， ratio = 1   
alpha   | N    | recall | precision | coverage  | popularity    | hit number
---     | ---  | ---    | ---       | ---       | ---           |  ---
0.02    | 10   | 6.16%  | 30.63%    | 28.28%    | 7.1996        | 18498
0.02    | 20   | 6.89%  | 34.22%    | 31.07%    | 7.1581        | 20671
0.015   | 10   | 5.62%  | 27.92%    | 25.11%    | 7.2905        | 16865
0.015   | 20   | 6.38%  | 31.67%    | 28.49%    | 7.2492        | 19131
F = 100, lambda = 0.01， ratio = 5   
alpha   | N    | recall | precision | coverage  | popularity    | hit number
---     | ---  | ---    | ---       | ---       | ---           |  ---
0.02    | 10   | 7.18%  | 35.66%    | 12.70%    | 7.1858        | 21539
0.02    | 20   | 1.29%  | 6.42%     | 56.57%    | 1.4448        | 3879
0.015   | 10   | 6.94%  | 34.50%    | 11.40%    | 7.2615        | 20835
0.015   | 20   | 7.86 % | 39.03%    | 16.14%    | 7.2084        | 23575

可见在ratio=1 的情况下得到的各个性能指标都已经达到了比较理想的状态，与之前基于邻域的算法得出的结果相差不多。估计如果继续优化各个参数，指标还会有进一步的提升。


*参考资料*
1. <span id="1"></span>推荐系统实践
2. <span id="2"></span>https://en.wikipedia.org/wiki/David_Wheeler_(British_computer_scientist)?oldformat=true
3. <span id="3"></span>http://zhangyi.space/ji-yu-yin-yu-yi-mo-xing-latent-factor-modelde-dian-ying-tui-jian-xi-tong-ji-qi-zai-sparkshang-de-shi-xian/
3. <span id="4"></span>http://blog.csdn.net/sinat_33741547/article/details/52976391

