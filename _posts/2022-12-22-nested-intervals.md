---
title: 树形结构数据存储之Nested Intervals
date: 2022-12-22 23:53:23 +0800
categories: [algorithm, python]
tags: [algorithm,design] 
toc: true
comments: true
math: true
mermaid: true
pin: true

---

<center>
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.
</center>

---

前段时间在开发中遇到一个在mysql中存储树形结构的数据的技术需求，由于数据量比较大层级多，且写入少查询多，常规的子母id映射的方法显然不大适用——因为查询次数太多了。所以调研了一下其他的方法。

在一位网友[@钱魏Way](https://www.biaodianfu.com/)的[这篇博文](https://www.biaodianfu.com/storing-trees-in-rdbms.html)里看到了一个总结文，介绍了几种设计方案，大部分都比较易于理解也各有优劣，其中的最后一种方式Nested Intervals令我饶有兴趣，一时没看明白，博主也表示还没搞懂从读取时将数值转换为结构的原理，按图索骥找到英文原文[Nested Sets and Materialized Path SQL Trees](http://www.rampant-books.com/art_vadim_nested_sets_sql_trees.htm)。打了几页草稿之后，终于弄明白了。

这种利用简单的四则计算从数值中解析出树形结构的设计，令某不经顿感惊为天人。mysql查询是毫秒级的，而这种计算几乎可以达到微秒级，效率的提升石破天惊。

## 1.基本设计

对照[原文](http://www.rampant-books.com/art_vadim_nested_sets_sql_trees.htm)，先介绍下该算法的基本设计

> Nested Intervals
> 
> Nested intervals generalize nested sets. A node [clft, crgt] is an (indirect) descendant of [plft, prgt] if:
> plft <= clft and crgt >= prgt
> The domain for interval boundaries is not limited by integers anymore: we admit rational or even real numbers, if necessary. Now, with a reasonable policy, adding a child node is never a problem. One example of such a policy would be finding an unoccupied segment [lft1, rgt1] within a parent interval [plft, prgt] and inserting a child node [(2*lft1+rgt1)/3, (rgt1+2*lft)/3]:
> 
> After insertion, we still have two more unoccupied segments [lft1,(2*lft1+rgt1)/3] and [(rgt1+2*lft)/3,rgt1] to add more children to the parent node.
> 
> We are going to amend this naive policy in the following sections.
> 
> Partial Order
> 
> Let's look at two dimensional picture of nested intervals. Let's assume that rgt is a horizontal axis x, and lft is a vertical one - y. Then, the nested intervals tree looks like this:
> 
> Each node [lft, rgt] has its descendants bounded within the two-dimensional cone y >= lft & x <= rgt. Since the right interval boundary is always less than the left one, none of the nodes are allowed above the diagonal y = x.
> The other way to look at this picture is to notice that a child node is a descendant of the parent node whenever a set of all points defined by the child cone y >= clft & x <= crgt is a subset of the parent cone y >= plft & x <= prgt. A subset relationship between the cones on the plane is a partial order.
> Now that we know the two constraints to which tree nodes conform, I'll describe exactly how to place them at the xy plane.
> 
> The Mapping
> 
> Tree root choice is completely arbitrary: we'll assume the interval [0,1] to be the root node. In our geometrical interpretation, all the tree nodes belong to the lower triangle of the unit square at the xy plane.
> We'll describe further details of the mapping by induction. For each node of the tree, let's first define two important points at the xy plane. The depth-first convergence point is an intersection between the diagonal and the vertical line through the node. For example, the depth-first convergence point for <x=1,y=1/2> is <x=1,y=1>. The breadth-first convergence point is an intersection between the diagonal and the horizontal line through the point. For example, the breadth-first convergence point for <x=1,y=1/2> is <x=1/2,y=1/2>.
> Now, for each parent node, we define the position of the first child as a midpoint halfway between the parent point and depth-first convergence point. Then, each sibling is defined as a midpoint halfway between the previous sibling point and breadth-first convergence point:
> For example, node 2.1 is positioned at x=1/2, y=3/8.
> Now that the mapping is defined, it is clear which dense domain we are using: it's not rationals, and not reals either, but binary fractions (although, the former two would suffice, of course).
> Interestingly, the descendant subtree for the parent node "1.2" is a scaled down replica of the subtree at node "1.1." Similarly, a subtree at node 1.1 is a scaled down replica of the tree at node "1." A structure with self-similarities is called a fractal.



先简述一下计算的方法，这是一个递归的方式：

1. 用数轴上的一个**区间**来代表一个节点P，那么这个区间可以表示为`[left,right]`，P有n个子节点，从左到右，分别编号为c1，c2，c3...

2. 处理子节点的规则是，
    - 2.1 从父节点P的区间里划出**右半区**用来表示P的第一个子节点c1，取left,right的平均值，设为`ave=(left+right)/2`，那么c1就表示为`[ave,right]`
    - 2.2 对于c1的紧接着的弟弟c2，我们从把c1的**左半区**用来表示，c1的中点设为 $ave1=(ave+right)/2$，即c2表示为`[ave,ave1]`
    
    总结就是，对任一个节点，都有表示该节点的一个区域，我们用它的左半区表示紧邻它的“弟弟”，用它的右半区表示它的“大儿子”即第一个子节点。

3. 接下来所有的都按照以上方法，递归地处理。

到这里，所有的策略都已经完备。

## 2.作图分析


再进一步，不失一般性，我们把根节点P设为`[0,1]`,那么 c1就是`[1/2,1]`, c2就是`[1/2,3/4]`......

此时我们把每个节点的区间表示左右调换，当成一个点的坐标画到二维空间中，就会得到这么一张图:

![img]({{ site.url }}/assets/img/source/nested_interval.png)

*图中 P1-2 代表第一层子节点的第二个*

geogebra原图
<iframe src="https://www.geogebra.org/geometry/zngv9rek?embed" width="800" height="600" allowfullscreen style="border: 1px solid #e4e4e4;border-radius: 4px;" frameborder="0"></iframe>

我们把刚才的递归策略结合这张图，用画图的方式再推演一次

0. 连接点`(0,0)`和点`(1,1)`，画线`AB`，这条线是一个辅助线；
1. 取点`(1,0)`为根节点`P0`；
2. 对于`P0`的第1个子节点，过`P0`做**平行于y轴**的直线交`AB`于`B`点，取`P0B`的中点为`P1-1`，此时`P1-1`坐标`(1,1/2)`就是`P0`的第一个子节点；
3. 对于`P0`的第2个子节点，此时不直接通过`P0`求出，而是利用上述2.2的规则通过`P1-1`求解，过`P1-1`做**平行于x轴**的直线交`AB`于`BC1`点，取`P1-1`和`BC1`的中点（坐标为$(3/4,1/2)$），该中点即为`P1-2`。

## 3.建模成果

如此往复，我们就可以在`A-B-P0`所围成的三角形内画出一棵无限大的树。不止于此，我们也可以按上述的规则3画出图中`P1`作为另一个根节点，那么就可以仅仅用`A-B-P0`三角形内的点集表示一个无限大的森林！

每个节点都由一个坐标表示，该坐标是两个分数，每个分数可以用两个自然数表示，那么我们用4个自然数，就可以表示一个节点，对于每一个节点，我们只需要在数据库里存储4个自然数，最重要的是，这四个自然数里含有该节点在树内的位置信息！

此外不难证明，任何一个节点的所有子节点（直接与间接）都在该节点与通过该节点的平行于坐标轴的两条线与AB交点围成的三角形中，距离，`P0`的所有子节点都在 `P0 - B - BC1` 三点围成的三角形中，且P0的所有直接子节点都在`P1-1 - BC1` 这条线段上。

## 4.反向解析

解决计算的问题后，接下来再看解析的问题，如何将节点的位置信息从2个分数（4个自然数）的坐标值中解析出来。

### 4.1 主要思路：使用坐标值的和，利用二分法确定

首先明确几个规律：

1. 易证区域内任何一个点的`x,y`坐标值的和，与该点到线段`AB`的垂足的坐标值的和相等，继而，每个点都可以映射为到`AB`上的垂足，如图中`P0`可以映射为`BC1`，`P1-1`可以映射为`BC2`，`P1-2-1`可以映射为点`p`
2. 每一个节点及其所有子节点都在 该节点 、 该节点与`AB`的垂足、点`B`所围成的三角形范围内，如`P1-1`对应图中粉色区域，`P1-2-1`对应蓝色区域，且该三角形属于且仅属于该节点。用分封制的概念来说，就是每个节点都会把自己三角形内的一部分互斥地分给自己的直接子节点，一层一层往下分封
3. 第2条的分封结合第1条的映射，可以进一步得到，每一个节点和自己的所有子节点，都会分得`AB`上的一段，然后会将自己的一段互斥地分给直接子节点。这里有个规律，每个子节点都会把自己在AB上的“领地”的上半段分给大儿子！
4. 所有的坐标值都是二分的，即分母一定是2的幂，因此我们可以根据坐标值的和，倒推出所在的节点


### 4.2 求解

倒推的步骤：

1. 把读取到的两个坐标值（分数）相加设为`x`，
2. 对比在AB上的相对位置（`A`点的坐标值和`0`，`B`点为`2`）
3. 计算AB中点的坐标值  $mid=(0+2)/2=1$
  - 3.1 如果$x=mid$，那么该点属于`P0`，是根节点，此时判定找到了节点位置，计算结束。
  - 3.2 如果$x<mid$，那么该点不属于`P0`所在的树
  - 3.3 如果$x>mid$，那么该点是`P0`的某个子节点（直接或非直接）
4. 此时将对比区域缩小。取`AB`的中点`M`，如果$x<mid$，继续对比`A`与`M`之间的区域，即`AM`，否则取`MB`
5. 重新对比计算`x`与`MB`中点的相对位置，如此往复，直到找到相等的值。


## 5. 代码实现

为了简化计算存储，每个坐标值的分数用一个两个int组成的tuple表示

根据输入的树形结构生成位置值的计算较为简单，此处略过，后续会放到本人的github中

（更新，代码段见 [gist](https://gist.github.com/buyiyihu/765a794ad4c9e40429c03ff28fa24328)）

根据位置值解析出相对位置
```python
from math import gcd

class Parser

    @classmethod
    def _sum_and_diff(cls, a: Tuple[int], b: Tuple[int]) -> Tuple[Tuple[int]]:
        # Calculate the sum and diffrence of two number formed by tuple
        gcd_value = gcd(a[1], b[1])
        ft_a = a[1] // gcd_value
        ft_b = b[1] // gcd_value
        denominator = a[1] * ft_b
        a_numerator = a[0] * ft_b
        b_numerator = b[0] * ft_a
        sumn = a_numerator + b_numerator
        sums = None
        if sumn % 2 == 1:
            sums = (sumn, denominator * 2), (sumn, denominator)
        numeraotr = sumn >> 1
        if sums is None:
            while numeraotr % 2 == 0:
                numeraotr >>= 1
                denominator >>= 1
            sums = (numeraotr, denominator), (numeraotr, denominator >> 1)
        return sums

    @classmethod
    def parse_position(cls, dp: Tuple[int], bd: Tuple[int]):
        _, (nomi, deno) = cls._sum_and_diff(dp, bd)
        structure = []
        # structure stores the node's position in the tree, 
        # each value in the list represents the relative position 
        # when walking from the root to the node  
        low, high, pos = 0, deno << 1, 0
        # low, high represent the up and down bound of the value
        # pos stands for the relative position of siblings 
        while True:
            mid = (low + high) >> 1
            if nomi > mid:
                # Value is bigger than mid, means it is the sub node 
                # of the current postion's node
                structure.append(pos)
                pos = 0
                low = mid
            elif nomi < mid:
                # Value is smallere than mid, means it is the sub node 
                # of the silbings of the current postion's node, 
                # so add the pos and keep searching.
                pos += 1
                high = mid
            else:
                # Equaling means the current position 
                # is exactly the node we are searching
                structure.append(pos)
                break
        return structure

```
其中类方法`_sum_and_diff`用于计算两个分数的和与中值，`parse_position`解析位置，返回一个不定长的list，list的值表示该位置在树的每一层的index，举个例子下图中节点11的位置用列表表示为： `[0,0,2,1]`.

![img]({{ site.url }}/assets/img/source/atree.png)




