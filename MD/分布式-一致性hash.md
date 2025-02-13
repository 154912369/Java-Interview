# 一致 Hash 算法
分布式缓存中，如何将数据均匀的分散到各个节点中，并且尽量的在加减节点时能使受影响的数据最少。

## Hash 取模
随机放置就不说了，会带来很多问题。通常最容易想到的方案就是 `hash 取模`了。

可以将传入的 Key 按照 `index = hash(key) % N` 这样来计算出需要存放的节点。其中 hash 函数是一个将字符串转换为正整数的哈希映射方法，N 就是节点的数量。

这样可以满足数据的均匀分配，但是这个算法的容错性和扩展性都较差。

比如增加或删除了一个节点时，所有的 Key 都需要重新计算，显然这样成本较高，为此需要一个算法满足分布均匀同时也要有良好的容错性和拓展性。

## 一致 Hash 算法

一致 Hash 算法是将所有的哈希值构成了一个环，其范围在 `0 ~ 2^32-1`。如下图：

![](../images/一致性hash1.jpg)

之后将各个节点散列到这个环上，可以用节点的 IP、hostname 这样的唯一性字段作为 Key 进行 `hash(key)`，散列之后如下：

![](../images/一致性hash2.jpg)

之后需要将数据定位到对应的节点上，使用同样的 `hash 函数` 将 Key 也映射到这个环上。

![](../images/一致性hash3.jpg)

这样按照顺时针方向就可以把 k1 定位到 `N1节点`，k2 定位到 `N3节点`，k3 定位到 `N2节点`。

### 容错性
这时假设 N1 宕机了：

![](../images/一致性hash4.jpg)

依然根据顺时针方向，k2 和 k3 保持不变，只有 k1 被重新映射到了 N3。这样就很好的保证了容错性，当一个节点宕机时只会影响到少少部分的数据。

### 拓展性

当新增一个节点时:

![](../images/一致性hash5.jpg)

在 N2 和 N3 之间新增了一个节点 N4 ，这时会发现受印象的数据只有 k3，其余数据也是保持不变，所以这样也很好的保证了拓展性。

## 虚拟节点
到目前为止该算法依然也有点问题:

当节点较少时会出现数据分布不均匀的情况：

![](../images/一致性hash6.jpg)

这样会导致大部分数据都在 N1 节点，只有少量的数据在 N2 节点。

为了解决这个问题，一致哈希算法引入了虚拟节点。将每一个节点都进行多次 hash，生成多个节点放置在环上称为虚拟节点:

![](../images/一致性hash7.jpg)

计算时可以在 IP 后加上编号来生成哈希值。

这样只需要在原有的基础上多一步由虚拟节点映射到实际节点的步骤即可让少量节点也能满足均匀性。

https://github.com/crossoverJie/JCSprout/blob/master/MD/Consistent-Hash.md

欢迎光临[我的博客](http://www.wangtianyi.top/?utm_source=github&utm_medium=github)，发现更多技术资源~
