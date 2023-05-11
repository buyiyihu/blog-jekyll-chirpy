---
title: Python字典与hash方法探究
date: 2020-08-20 21:52:33 +0800
categories: [Python]
tags: [algorithm,design,python] 
toc: true
comments: true
math: true
mermaid: true
pin: true
---

<center>
<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/80x15.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.
</center>

----


*看python的官方文档注意到字典的一些设计机理，特地查阅整理了一下关于python字典的特性、相关的hash方法的资料，并作此文记录。示例代码都是手敲的，文中也还有一些疑问尚未解决，欢迎讨论。转发请注明出处。文以析理而文责则负，笔者才疏学浅文中不免有错误疏漏之处，如有不及之处还请方家指正*



----

![img]({{ site.url }}/assets/img/source/hash_table.svg)

### 1. Python字典的设计


#### 1.1 基本原理

Python的字典使用了hash table[^refer-1]，当hash值碰撞时，利用open addressing[^refer-2]<sup>,</sup>[^refer-3]策略解决，即按照某种策略反复探测直到找到空的bucket再将数据插入。Python实现的具体策略网上没有太多明确的资料，看到的资料也有许多已经过时。官方文档也没有太多的原理介绍，大都是介绍如何使用。个人猜测，可能是具体的实现一直在变化，除非遇到问题需要debug可以直接看python源码，所以没必要在文档中详述实现。


hash table 的时间性能分析[^refer-4]：

| Algorithm | Average | Worst case |
| --------- | ------- | ---------- |
| Space     | $O(n)$    | $O(n)$       |
| Search    | $O(1)$    | $O(n)$       |
| Insert    | $O(1)$    | $O(n)$       |
| Delete    | $O(1)$    | $O(n)$       |

此处的最差情况，出现在hash碰撞时，需要通过probe（探测）重新定位，造成线性时间消耗。



#### 1.2 有序性

Python在3.5（含）以前，dict类型是无序的，即读取时的顺序不一定与写入的顺序一致。具体原理可以参考这个stackoverflow问题[^refer-5]。
简要总结一下，旧版的dict使用一个二维数组存储，创建时会创建一个较小的二维数组，写入时新键值对时，在得到hash值之后再对数组长度取模，以该取模值作为index值，将hash值，键的引用，值的引用作为一个一维数组写入二维数组中该index值的位置，当空间占用达到预分配的2/3时，会resize增加空间。可见写入的index源自hash值，与先后顺序没有关系。遍历读取时，直接遍历该二维数组，此时显然输出的顺序与写入的顺序没有关系。

新版的dict，在二维数组之上加了个列表，该列表承担了hash table的功能，即下标对应键的hash值的取模结果，而存储的值是该条数据（hash值，键的引用，值的引用）在二维数组中的index。

相当于将hash table的功能剥离出来放在这个列表中，再去映射到二维数组。这样一来，一方面不必预先分配多余空间给二维数组，节省了空间。官方文档表示3.6比3.5的dict少用20%～25%的内存[^refer-6]：


> The memory usage of the new dict() is between 20% and 25% smaller compared to Python 3.5.



另一方面，显然，此时二维数组相当于一个链表，随用随加，不必预先分配大量空间，也保持了插入的顺序。在全量读取时，直接遍历该二维数组即可顺序输出。另外由于该二维数组中间没有空档，所以也减少了遍历时的次数。

以下是来自该方案提出者Raymond Hettinger的一个例子和解释[^refer-7]：


> Instead, the 24-byte entries should be stored in a dense table referenced by a sparse table of indices.
> 
> For example, the dictionary:
> 
>     d = {'timmy': 'red', 'barry': 'green', 'guido': 'blue'}
> 
> is currently stored as:
> 
>     entries = [['--', '--', '--'],
>                [-8522787127447073495, 'barry', 'green'],
>                ['--', '--', '--'],
>                ['--', '--', '--'],
>                ['--', '--', '--'],
>                [-9092791511155847987, 'timmy', 'red'],
>                ['--', '--', '--'],
>                [-6480567542315338377, 'guido', 'blue']]
> 
> Instead, the data should be organized as follows:
> 
>     indices =  [None, 1, None, None, None, 0, None, 2]
>     entries =  [[-9092791511155847987, 'timmy', 'red'],
>                 [-8522787127447073495, 'barry', 'green'],
>                 [-6480567542315338377, 'guido', 'blue']]
> 
> Only the data layout needs to change.  The hash table algorithms would stay the same.  All of the current optimizations would be kept, including key-sharing dicts and custom lookup functions for string-only dicts.  There is no change to the hash functions, the table search order, or collision statistics.
> 
> The memory savings are significant (from 30% to 95% compression depending on the how full the table is). Small dicts (size 0, 1, or 2) get the most benefit.



该特性从python3.6成为dict的实现，从3.7开始正式成为一个feature

#### 1.3 字典查询操作的细节


内建方法hash调用传入对象的`__hash__`方法生成并返回该对象的hash值，如果该对象可哈希并有hash值的话。这些hash值都是整数，因此`__hash__`方法如果返回的话必须返回一个int，用于字典查找时对比键。

字典查询时大致有三步：
1. 计算键的hash值，并去hash table中寻找
2. 如果字典中存在对应该hash值的键，则依次对比这些同hash值的键。首先对比二者是否为同一个对象，是则查询结束
3. 如果不是同一个对象，则对比二者是否相同，是则查询结束
4. 对比完所有同hash的键后都不符合，则判定不存在该键

一下用一个特殊设计的类来验证演示（python3.9）：
```
In [1]: class A:   
   ...:     def __init__(self,a,b):   
   ...:         self.a=a   
   ...:         self.b=b   
   ...:     def __eq__(self,ot):   
   ...:         print(f">>> comparing   self:{self} vs other:{ot}")   
   ...:         return self.a==ot.a   
   ...:     def __hash__(self): 
   ...:         hash_value = hash(self.b) 
   ...:         print(f">>> hashing  {self} -> {hash_value}")   
   ...:         return hash_value 
   ...:     def __repr__(self):   
   ...:         return f"<A({self.a},{self.b})[{id(self)%100000000}]>"
   ...:         

In [2]: a,b,c,d,e = [A(i,'a') for i in range(1,6)]
In [3]: s = {a:1}      
>>> hashing  <A(1,a)[3154304]> -> 5644554256352420145
```
新建字典或插入不存在对应hash值新键时会先计算键的hash值

```
In [4]: s[b]=1        
>>> hashing  <A(2,a)[3154360]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(2,a)[3154360]>
```
新增新键时会先计算键的hash值，如果发现已经存在一样hash值的键，会对进行判等比较计算

```
In [5]: aa = A(1,'a')
In [6]: s[aa]=11
>>> hashing  <A(1,a)[3057568]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(1,a)[3057568]>

n [7]: s[aa]
>>> hashing  <A(1,a)[3057568]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(1,a)[3057568]>
Out[7]: 11

In [8]: s[a]
>>> hashing  <A(1,a)[3154304]> -> 5644554256352420145
Out[8]: 12

```
如果两个对象的hash值一致，会先判断是否为同一个对象，如果不是，进行等于计算，结果是True的话，那么在字典里查询时会当成是同一个键。并且不会用新的表量替换掉键的旧变量。

```
In [9]: s[c]=1         
>>> hashing  <A(3,a)[3154416]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(3,a)[3154416]>
>>> comparing   self:<A(2,a)[3154360]> vs other:<A(3,a)[3154416]>

```
此时插入第3个hash值相等键，新键会和hash值相等键依次对比。查询同理

```
In [10]: s[d]=1
>>> hashing  <A(4,a)[3154472]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(4,a)[3154472]>
>>> comparing   self:<A(2,a)[3154360]> vs other:<A(4,a)[3154472]>
>>> comparing   self:<A(3,a)[3154416]> vs other:<A(4,a)[3154472]>
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(4,a)[3154472]>

```
此时插入第4个hash值相等键，依旧，新键会和hash值相等键依次对比。但此时和其中的一个键对比了两次。这一情况不能完全复现，在python3.6,3.7,3.8,3.9中都有出现。猜测可能是probe的顺序，这些内部实现的细节有关，没有找到相关资料

```
In [11]: s[e]=1
>>> hashing  <A(5,a)[3154584]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(2,a)[3154360]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(3,a)[3154416]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(4,a)[3154472]> vs other:<A(5,a)[3154584]>
```

此时插入第5个hash值相等键，依旧，新键会和hash值相等键依次对比。并且，新键插入会重复之前的所有对比计算，对比了两次的键`d`，此时依然对比了两次

```
In [12]: s[a]
>>> hashing  <A(1,a)[3154304]> -> 5644554256352420145
Out[12]: 1

In [13]: s[b]
>>> hashing  <A(2,a)[3154360]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(2,a)[3154360]>
Out[13]: 1

In [14]: s[c]
>>> hashing  <A(3,a)[3154416]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(3,a)[3154416]>
>>> comparing   self:<A(2,a)[3154360]> vs other:<A(3,a)[3154416]>
Out[14]: 1

In [15]: s[d]
>>> hashing  <A(4,a)[3154472]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(4,a)[3154472]>
>>> comparing   self:<A(2,a)[3154360]> vs other:<A(4,a)[3154472]>
>>> comparing   self:<A(3,a)[3154416]> vs other:<A(4,a)[3154472]>
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(4,a)[3154472]>
Out[15]: 1

In [16]: s[e]
>>> hashing  <A(5,a)[3154584]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(2,a)[3154360]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(3,a)[3154416]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(4,a)[3154472]> vs other:<A(5,a)[3154584]>
Out[16]: 1


In [17]: s
Out[17]: >>> hashing  <A(1,a)[3154304]> -> 5644554256352420145
>>> hashing  <A(2,a)[3154360]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(2,a)[3154360]>
>>> hashing  <A(3,a)[3154416]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(3,a)[3154416]>
>>> comparing   self:<A(2,a)[3154360]> vs other:<A(3,a)[3154416]>
>>> hashing  <A(4,a)[3154472]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(4,a)[3154472]>
>>> comparing   self:<A(2,a)[3154360]> vs other:<A(4,a)[3154472]>
>>> comparing   self:<A(3,a)[3154416]> vs other:<A(4,a)[3154472]>
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(4,a)[3154472]>
>>> hashing  <A(5,a)[3154584]> -> 5644554256352420145
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(2,a)[3154360]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(3,a)[3154416]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(1,a)[3154304]> vs other:<A(5,a)[3154584]>
>>> comparing   self:<A(4,a)[3154472]> vs other:<A(5,a)[3154584]>

{<A(1,a)[3154304]>: 1,
 <A(2,a)[3154360]>: 1,
 <A(3,a)[3154416]>: 1,
 <A(4,a)[3154472]>: 1,
 <A(5,a)[3154584]>: 1}


```
输出字典的所有内容，会依次计算每个键的hash，找到对应的value。查询所有数据是查询单个键值对的操做的简单累加，如果hash一致依然会接着做等于计算。
hash值相同的键之间有一种“排序”机制，获取键值时会依次对比直到找到为止。为什么用“排序”这个词，因为输出时，顺序是和插入一致的，但是对比时却不一定，并且调用排python解释器执行的结果也不尽相同。另外在python2.7中，输出所有键值对时，是没有做hash计算的。只能说底层的实现一直在变。

完整代码
```
class A:
    def __init__(self,a,b):
        self.a=a
        self.b=b
    def __eq__(self,ot):
        print(f">>> comparing   self:{self} vs other:{ot}")
        return self.a==ot.a
    def __hash__(self):
        hash_value = hash(self.b)
        print(f">>> hashing  {self} -> {hash_value}")
        return hash_value
    def __repr__(self):
        return f"<A({self.a},{self.b})[{id(self)%100000000}]>" 
       
a,b,c,d,e = [A(i,'a') for i in range(1,6)]
aa = A(1,'a')
s = {a:1}
s[aa] = 10
s[b] = 2
s[c] = 3
s[d] = 4
s[e] = 5
s
```

----

### 2. CPython `__hash__` 的实现算法

#### 2.1 基本参数


hash值的计算的有关参数在`sys.hash_info`中[^refer-8]：
```
python -c "import sys;print(sys.hash_info)"
sys.hash_info(
    width=64, modulus=2305843009213693951, 
    inf=314159, nan=0, imag=1000003, 
    algorithm='siphash24', hash_bits=64, 
    seed_bits=128, cutoff=0
    )
```

|  attribute | explanation  |
|-----------|---------------|
| width | width in bits used for hash values |
| modulus | prime modulus P used for numeric hash scheme | 
| inf  | hash value returned for a positive infinity |
| nan | hash value returned for a nan |
| imag | multiplier used for the imaginary part of a complex number |
| algorithm | name of the algorithm for hashing of str, bytes, and memoryview |
| hash_bits | internal output size of the hash algorithm |
| seed_bits | size of the seed key of the hash algorithm | 


`__hash__()`方法必须返回一个int类型，解释器会把该方法的返回的值裁剪到 `Py_ssize_t`的大小，该值在64位机器上是8字节，32位机器上是4字节[^refer-9]。以64位机器为例，鉴于返回值是有符号的64位整数，范围为$[-2^{63},2^{63}-1]$,也就是 $[-2^{sys.hash\_info.width - 1}$,$2^{sys.hash\_info.width - 1})$ 超过该范围会自动截断。如以下下示例：

```
In [1]: b = 2**63-1 
In [2]: class B: 
   ...:     def __init__(self,x): 
   ...:         self.x = x 
   ...:     def __hash__(self): 
   ...:         return b + self.x 

In [3]: for i in range(10): 
   ...:     print(f"hashing>> original:2^63-1+{i}  actual:{hash(B(i))}") 
   ...           
hashing>> original:2^63-1+0  actual:9223372036854775807
hashing>> original:2^63-1+1  actual:4
hashing>> original:2^63-1+2  actual:5
hashing>> original:2^63-1+3  actual:6
hashing>> original:2^63-1+4  actual:7
hashing>> original:2^63-1+5  actual:8
hashing>> original:2^63-1+6  actual:9
hashing>> original:2^63-1+7  actual:10
hashing>> original:2^63-1+8  actual:11
hashing>> original:2^63-1+9  actual:12
```

python中对于不同的数据类型，hash值的产生有不同产生设计。下面分4类讨论，数字类型（int,float,bool,complex）,字符串和bytes，特殊类类型以及自定义的类。

#### 2.2 数字类型hash算法

所有数字类型（numeric types）包括int, float, decimal.Decimal，fractions.Fraction，complex，只要值相等即`x==y`，则必有`hash(x)==hash(y)`

数字类型的hash值计算需要一个大质数用于取模计算。在当前CPython（python3.9）的实现中，使用的大质数，在32位机器上是 $2^{31}-1 =2147483647$ ，在64位机器上是 $2^{61}-1=2305843009213693951$

以下是来自英文版官方文档[^refer-10]的具体规则（没有使用中文版本是因为个别地方似乎用了机翻有误）：<br>
**1)**
> 如果$x=m/n$ 是一个非负有理数且$n$不可被$P$整除,则定义$hash(x)=m\times invmod(n,P)modP$, 其中 $invmod(n,P)$ 是 $n$ 模 $P$的模逆元。
  
对于所有小于$P=2^{61}-1=2305843009213693951$ 的非负整数，可取$n=1$，则有$n$不可被$P$整除，而此时invmod(n,P)可取$1$，则有$m\times invmod(n,P)=m\times 1=m$，$hash(x)=m modP=m$，也就是所有小于$2^{61}-1$的非负整数的hash值都等于其自身。也就是符合该项要求的非负有理数的hash值最大只有$2^{61}-2$，然后从$0$继续递增。

示例：
```
In [1]: P = 2**61-1
In [2]: hash(P-1)==P-1
Out[2]: True
In [3]: for i in range(10):
   ...:     print(f"int:2**61-1+{i} -> hash:{hash(P+i)}")
   ...:
int:2**61-1+0 -> hash:0
int:2**61-1+1 -> hash:1
int:2**61-1+2 -> hash:2
int:2**61-1+3 -> hash:3
int:2**61-1+4 -> hash:4
int:2**61-1+5 -> hash:5
int:2**61-1+6 -> hash:6
int:2**61-1+7 -> hash:7
int:2**61-1+8 -> hash:8
int:2**61-1+9 -> hash:9
```

**2)**
> 如果$x=m/n$ 是一个非负有理数且有$n$可被$P$整除而$m$不可，此时因为$n,P$不互质，则$n$以$P$为模无模逆元，上述1项不适用，此时定义 `hash(x)` 值为常数值 `sys.hash_info.inf`


构造一个案例验证一下：
```
In [1]: import sys
In [2]: from fractions import Fraction as F
In [3]: P = 2**61-1
In [4]: all(map(lambda x:hash(x)==sys.hash_info.inf,(F(i,P) for i in range(1,10000))))
Out[4]: True
```
float类型由于丢失精度，不再成立
```
In [5]: hash(1/P)
Out[5]: 1
In [5]: hash(1/(P-1))
Out[5]: 1
In [6]: hash(2/P)
Out[6]: 2
```

**3)**
> 如果$x=m/n$ 是一个负有理数，则 $hash(x)= -hash(-x)$, 且若值为$-1$ 则替换为 $-2$

此项操作相当于把负有理数转换为正数后在通过1),2)步进行计算

**4)**
> 特定值 `sys.hash_info.inf`, `-sys.hash_info.inf` 与 `sys.hash_info.nan`分别作为$+\infty$，$-\infty$，`NaN`的hash值，所有的`NaN`有着一样的hash值


示例：
```
In [1]: import sys
In [2]: sys.hash_info.inf
Out[2]: 314159
In [3]: sys.hash_info.nan
Out[3]: 0
In [4]: inf = float('inf')
In [5]: nan = float('nan')
In [6]: hash(inf)
Out[6]: 314159
In [7]: hash(nan)
Out[7]: 0
In [8]: hash(nan-nan)
Out[8]: 0
```

**5)**
> 对于一个复数$z$,实部和虚部的hash值通过公式$hash(z.real) + sys.hash\_info.imag \times hash(z.imag)$合到一起，再对 $2^{sys.hash\_info.width}$ 取模，最终整体hash值落在 $[-2^{sys.hash\_info.width - 1}$,$2^{sys.hash\_info.width - 1})$ 之间。同样，且若值为$-1$ 则替换为 $-2$

此处的取模做了特殊处理，对 $2^{sys.hash\_info.width}$ 取模，但由于是有符号数，最终取$[-2^{sys.hash\_info.width - 1}$,$2^{sys.hash\_info.width - 1})$之间的值。


#### 2.3 字符串和bytes的hash算法

对字符串和bytes类型的hash值计算略微复杂一些，主要是通过各种位操作加上随机盐值使用hash算法生成。

默认情况下，python解释器会使用一个随机数作为hash种子。该随机数会用于str和bytes对象的hash值生成时加盐。这会导致不同python进程之间，同一个str和bytes对象的hash值时是否一样无法预测，但同一个进程内部，还是会保持一致的。

#### 2.4 特殊类型的hash算法

对于tuple类型，hash值的计算是将各个元素的hash值经过略微复杂的位计算合为一个。所以，tuple类型的hash值与内部元素的内容和id都没有直接关系，只和它们的hash值有关系[^refer-11]。这也是为什么官方文档也推荐自定义生成类的hash值时，使用主要属性组成的tuple的hash值[^refer-9]：


>  it is advised to mix together the hash values of the components of the object that also play a part in comparison of objects by packing them into a tuple and hashing the tuple


```
In [30]: hash((1,[1,]))
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-30-69711a6cc904> in <module>
----> 1 hash((1,[1,]))

TypeError: unhashable type: 'list'

```
此处报错指出，list类型不可hash，而不是说这个tuple不可hash，因为在计算tuple的hash时，首先要计算每一个子项的hash值

此外datetime类型的对象也有其独特的计算方法。

#### 2.5 自定义类的hash算法

对于自定义类，hash值的产生来自`id()`[^refer-16]：

>  Objects which are instances of user-defined classes are hashable by default. They all compare unequal (except with themselves), and their hash value is derived from their id().


有人指出[^refer-12]以往python中的实现是 `id`值除以16，即：
$hash(a)=id(a)\div16$，在3.9中实测发现并不全是这样。当hash值是正值时，一般符合此规律，而为负值时不然。见以下案例：
```
In [1]: class A:
   ...:     pass

In [2]: objs = [A() for i in range(10)]

In [3]: for o in objs:
   ...:     print(f"Compare: id:{id(o)} hash:{hash(o)}, 16times?:{id(o)==hash(o)*16}")
Compare: id:139796198843504 hash:8737262427719,             16times?:True
Compare: id:139796198843728 hash:8737262427733,             16times?:True
Compare: id:139796198844176 hash:8737262427761,             16times?:True
Compare: id:139796198844288 hash:8737262427768,             16times?:True
Compare: id:139796198843672 hash:-9223363299592348079,      16times?:False #X
Compare: id:139796198843896 hash:-9223363299592348065,      16times?:False #X
Compare: id:139796198843952 hash:8737262427747,             16times?:True
Compare: id:139796198844064 hash:8737262427754,             16times?:True
Compare: id:139796198844568 hash:-9223363299592348023,      16times?:False #X
Compare: id:139796198844624 hash:8737262427789,             16times?:True

```
这是因为做了一定的随机化处理，但大原则依然不变——依赖于该对象在内存中的位置。这也确保对象在其生命周期内，不会与其他自定义对象的hash值碰撞。而对于内建类型的hash值，还是有碰撞的可能，但是还有`__eq__`的机制。


#### 2.6 hash碰撞攻击和安全问题

##### 2.6.1 hash 种子

默认情况下，python解释器会使用一个随机数作为hash种子用于str和bytes对象的hash值生成时加盐。
此功能可以通过环境变量`PYTHONHASHSEED`来调整[^refer-13]。

* 若该变量未设置或设置为random，一个随机值会作为种子来生成str和bytes类型的hash值；
* 若设为一个int值，该值会作为hash随机所涉及到的类型的hash值生成时的固定种子值。
* 该值必须是一个[0,4294967295]区间内的decimal数值，设为0则会关闭hash随机功能

该值的用处是实现可重现的hash，如在解释器测试或python进程集群时共享一致的hash值。


##### 2.6.2 以hash碰撞造成拒绝服务攻击

hash碰撞会导致字典的操作处于时间复杂度最差情况，造成负载过高而DoS.
参见：
* [Huge portions of the Web vulnerable to hashing denial-of-service attack](https://arstechnica.com/information-technology/2011/12/huge-portions-of-web-vulnerable-to-hashing-denial-of-service-attack/)

* [Denial of service via hash collisions](https://lwn.net/Articles/474912/)

* [What Exactly is Hash Collision](https://stackoverflow.com/questions/45795637/what-exactly-is-hash-collision)



----


### 3. python的`__hash__`协议


#### 3.1 hashable

hashable，即可哈希类型，是指在该对象的生命周期内有一个**不变**的hash值，该类形对象必须有`__hash__`方法，且必须可比较，即有`__eq__()`方法——因为hash碰撞时需要做对比。

这是hashable的定义和原理，而可哈希（hashable）的概念和不可变类型（mutable）悉悉相关。
mutable指对象可以在保持`id()`不变时改变值[^refer-14]，immutable的值是固定的不可改变，改变值时需要新建一个对象来存储[^refer-15]。然而对于自定义类，这一点从语法上不是强制的，但是是一条重要规范，此处的mutable是一种语义概念而不是python语法概念。从语义概念的角度解释，既然可变，那么该类的组成部分就可以修改，纵然可以人为地使其hash值不变，但这种对应的解体显然是不合理的。


问题来了，Python解释器运行时如何判断一个对象是否是hashable/mutable的？并不知道，也是依靠`__hash__`方法来判断的。如果一个对象的`__hash__`为`None`，则该对象为unhashable，也就不能作为字典的键，语义上也就等同于mutable。示例：
```
In [24]: hash([1,2])   
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-24-9ce67481a686> in <module>
----> 1 hash([1,2])
TypeError: unhashable type: 'list'

In [25]: class B: 
    ...:     __hash__ = None 
In [26]: b = B()
In [27]: hash(b)
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-27-ad85d8b55702> in <module>
----> 1 hash(b)
TypeError: unhashable type: 'B'

In [28]: s = {b:1}
---------------------------------------------------------------------------
TypeError                                 Traceback (most recent call last)
<ipython-input-28-3396bea44070> in <module>
----> 1 s = {b:1}
TypeError: unhashable type: 'B'
```
当然此处类`B`依然是hashable的，这设计到元类的问题，此处不表。
```
In [29]: s = {B:1}
In [30]:  

```

另外，可hash对象如果对比时相等则必须有一样的hash值。这是官方文档的要求[^refer-16]。这一点在语法上是不强制的，参考如下示例：
```
In [1]: class A: 
    ...:     def __init__(self,a,b): 
    ...:         self.a = a 
    ...:         self.b = b 
    ...:     def __eq__(self,ot): 
    ...:         return self.a == ot.a 
    ...:     def __hash__(self): 
    ...:         return hash(self.b)
In [2]: a = A(1,'a')
In [3]: b = A(1,'b')
In [4]: a == b
Out[4]: True
In [5]: s = {}
In [6]: s[a] = 1
In [7]: s[b] = 2
In [8]: s
Out[8]: {<__main__.A at 0x7fabe8de10b8>: 1, <__main__.A at 0x7fabe8c69f28>: 2}
```

python内建对象是都符合这个要求的。基本类型上面已经讨论过，自定义对象的默认`__eq__()`其实就是`is`计算，即如果不改写，自定义对象只有是同一个对象时才相等。
为什么要在语法要求外对自定义对象加上这一条要求，是因为，hash值是允许相等的，hash碰撞很难避免其实也没必要完全避免，只要足够分散平均就好，而`__eq__()`是避免字典不出问题的最后一道保障。另外从语义上说，两个对象都相等了，为什么区分度更低的hash值还不相等？两个相等的键在字典里却有两个不同的值，这是极其容易引起误解和bug的。

#### 3.2 `__hash__` 与`__eq__`

python的`__hash__` 方法与`__eq__`方法悉悉相关，官方文档对此二方法有明确规范[^refer-9]。
> If a class does not define an `__eq__()` method it should not define a `__hash__()` operation either; if it defines `__eq__()` but not `__hash__()`, its instances will not be usable as items in hashable collections. If a class defines mutable objects and implements an `__eq__()` method, it should not implement `__hash__()`, since the implementation of hashable collections requires that a key’s hash value is immutable (if the object’s hash value changes, it will be in the wrong hash bucket).
>
> User-defined classes have `__eq__()` and `__hash__()` methods by default; with them, all objects compare unequal (except with themselves) and x.`__hash__()` returns an appropriate value such that x == y implies both that x is y and hash(x) == hash(y).
>
> A class that overrides `__eq__()` and does not define `__hash__()` will have its `__hash__()` implicitly set to None. When the `__hash__()` method of a class is None, instances of the class will raise an appropriate TypeError when a program attempts to retrieve their hash value, and will also be correctly identified as unhashable when checking isinstance(obj, collections.abc.Hashable).
>
> If a class that overrides `__eq__()` needs to retain the implementation of `__hash__()` from a parent class, the interpreter must be told this explicitly by setting` __hash__` = `<ParentClass>.__hash__`.
>
> If a class that does not override `__eq__()` wishes to suppress hash support, it should include` __hash__` = None in the class definition. A class which defines its own `__hash__()` that explicitly raises a TypeError would be incorrectly identified as hashable by an `isinstance(obj, collections.abc.Hashable)` call.

总结以上官方文档所述：
<table>
    <tr>
        <th colspan="2" rowspan="2">规范</th><th colspan="2"><strong><code>__hash__()</code></strong> </th>
    </tr>
    <tr>
        <td>有</td><td>无</td>
    </tr>
    <tr>
        <th rowspan="2" ><strong><code>__eq__()</code></strong></th>
        <td>有</td>
        <td>可以！<br>此时该类型是一个hashable类型<br>用户自定义的类默认有此二方法，且都依赖于<code>id()</code>:<br>hash值来自于<code>id()</code><br>是否相等此时就是<code>is</code>计算,也取决于<code>id()</code></td>
        <td>没问题。<br>会被当成unhashable，不能作为字典的键。<br>对于自定义的mutable类型，应该设置成这种状态<br></td>
    </tr>
    <tr>
        <td>无</td>
        <td>不可以！<br>python解释器会当成hashable，但不足以作为字典的键，会导致bug<br>
        但这种情况一般不会出现，因为自定的类会继承而自带默认方法
        </td>
        <td>用户定义的类自带默认的内建方法</td>
    </tr>
    <tr><th>重写的规则</th>
    <td colspan="4">
    <ol>
    <li>仅重写<code>__eq__</code>，则<code>__hash__()</code>会自动赋为<code>None</code></li>
    <li>此时如果希望保留父类的<code>__hash__</code>则需要手动赋值 <code>__hash__= &lt;ParentClass&gt;.__hash__</code></li>
    <li>不重写<code>__eq__</code>而需要取消hash支持，需要手动将<code>__hash__</code>赋为<code>None</code></li>
    </ol>
    <strong>*</strong> <code>__hash__</code>为<code>None</code>时，对类实例进行hash会抛出<code>TypeError</code>而自动判定为unhashable<br>
    <strong>*</strong> 自行抛出<code>TypeError</code>不会判定为unhashable，仅当成普通报错</td></tr>
</table>



或者下表[^refer-17]这样：

| `__eq__()`              | `__hash__()`              | Description                                                                                                                                                          |
| ----------------------- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Defined<br>(by default) | Defined<br>(by default)   | If left as is, all objects compare unequal <br>(except themselves)                                                                                                   |
| (If mutable)<br>Defined | Should not <br>be defined | Implementation of hashable collection<br> requires key's hash value be immutable                                                                                     |
| Not defined             | Should not <br>be defined | If `__eq__()` isn't defined,<br> `__hash__()` should not be defined.                                                                                                 |
| Defined                 | Not defined               | Class instances will not be usable as hashable collection.<br> `__hash__()` implicity set to `None`.<br> Raises `TypeError` exception if tried to retrieve the hash. |
| Defined                 | Retain from Parent        | `__hash__ = <ParentClass>.__hash__`                                                                                                                                  |
| Defined                 | Doesn't want to hash      | `__hash__ = None`.<br> Raises `TypeError` exception <br>if tried to retrieve the hash.                                                                               |


#### 3.3 自定义实现的一般实践


如果需要特殊设定，较好的方法，是以类的关键属性值组成的tuple的hash值[^refer-20]。该方法速度差不多与其他hash实现一样快。小型的tuple的生成被特别优化过，其hash值的生成通过C builtins,比python代码要快[^refer-18]。

现在举个例子：
有个自定义类`A`：
```
class A(object):

    def __init__(self, a, b, c):
        self._a = a
        self._b = b
        self._c = c

    def __eq__(self, othr):
        return (isinstance(othr, type(self))
                and (self._a, self._b, self._c) ==
                    (othr._a, othr._b, othr._c))

    def __hash__(self):
        return hash((self._a, self._b, self._c))
```
显然`A`的实例与相同三个参数组成的tuple具有相同的hash值

```
>>> a = A(1, 2, 3)
>>> b = 1, 2, 3
>>> hash(a)==hash(b)
True
```
但是二者作为字典的键时，却不会冲突：
```
>>> s = {a: 123}
>>> a in s
True
>>> b in s 
False
```


这里[^refer-19]有个关于hash值不一致的问题很好的例子，在这种情况下，应该用`id()`来产生hash值。

----



### 参考资料


[^refer-1]: [Hash table at Wikipeida](https://en.wikipedia.org/wiki/Hash_table)

[^refer-2]: [Hash table at Wikipeida](https://en.wikipedia.org/wiki/Hash_table#Open_addressing)

[^refer-3]: [Open addressing at Wikipeida](https://en.wikipedia.org/wiki/Open_addressing)

[^refer-4]: [Time complexity in big O notation](https://en.wikipedia.org/wiki/Hash_table)

[^refer-5]: [Are dictionaries ordered in Python 3.6+?](https://stackoverflow.com/questions/39980323/are-dictionaries-ordered-in-python-3-6)

[^refer-6]: [New dict implementation at Python documentation](https://docs.python.org/3.6/whatsnew/3.6.html#new-dict-implementation)

[^refer-7]: [More compact dictionaries with faster iteration](https://mail.python.org/pipermail/python-dev/2012-December/123028.html)

[^refer-8]: [`sys.hash_info` at Python documentation](https://docs.python.org/3.9/library/sys.html#sys.hash_info)

[^refer-9]: [Python documentation](https://docs.python.org/3/reference/datamodel.html#object.__hash__)

[^refer-10]: [Hashing of numeric types at Python documentation](https://docs.python.org/3.9/library/stdtypes.html#numeric-hash)

[^refer-11]: [Błażej Michalik's answer at stackoverflow](https://stackoverflow.com/questions/49722196/how-does-python-compute-the-hash-of-a-tuple/49722254#49722254)

[^refer-12]: [Duncan's answer at stackoverflow](https://stackoverflow.com/questions/11324271/what-is-the-default-hash-in-python/11324771#11324771)

[^refer-13]: [PYTHONHASHSEED at Python documentation](https://docs.python.org/3.9/using/cmdline.html#envvar-PYTHONHASHSEED)

[^refer-14]: [mutable at python documentation](https://docs.python.org/3/glossary.html#term-mutable)

[^refer-15]: [immutablein at python documentation](https://docs.python.org/3/glossary.html#term-immutable)

[^refer-16]: [hashable at python documentation](https://docs.python.org/3/glossary.html#term-hashable)

[^refer-17]: [Python `hash()` in programiz](https://www.programiz.com/python-programming/methods/built-in/hash)

[^refer-18]: [ShadowRanger's comment at stackoverflow](https://stackoverflow.com/questions/2909106/whats-a-correct-and-good-way-to-implement-hash/2909119#comment103700610_2909119)

[^refer-19]: [Jaswant P's question at stackoverflow](hhttps://stackoverflow.com/questions/61118043/python-dict-getk-returns-none-even-though-key-exists)

[^refer-20]: [What's a correct and good way to implement ``__hash__()``?](https://stackoverflow.com/questions/2909106/whats-a-correct-and-good-way-to-implement-hash)



