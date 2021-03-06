# 在Solidity中如何优化Gas第一部分：变量

![](https://img.learnblockchain.cn/2020/09/03/15991036968686.jpg)

*本文基于Solidity 0.5.8版本*

Gas优化是开发以太坊智能合约所面临的一个独特挑战。要想成功，我们需要学习solidity如何在幕后处理变量和函数。

因此我们将Gas优化分为两部分

在第一部分中，我们通过学习如何权衡变量打包和数据类型。

在第二部分中，我们通过学习可见性、减少执行和减少字节码来优化Gas。

我们所介绍的一些技术将可能违反众所周知的代码模式。在优化之前，我们应该始终考虑可能产生的技术债务和维护成本。

# 优化变量

## 变量打包

Solidity合约用连续32字节的插槽来储存。当我们在一个插槽中放置多个变量，它被称为变量打包。

变量打包就像俄罗斯方块游戏。如果我们试图打包的变量超过当前槽的32字节限制，它将被存储在一个新的插槽中。我们必须找出哪些变量最适合放在一起，以最小化浪费的空间。

因为使用每个插槽都需要消耗Gas，变量打包通过减少合约要求插槽数量，帮助我们优化Gas的使用。

我们来看个例子

```
uint128 a;
uint256 b;
uint128 c;
```

这些变量无法打包。如果`b`和`a`打包在一起，那么就会超过32字节的限制，所以会被放在新的一个储存插槽中。`c`和`b`打包也如此。

```
uint128 a;
uint128 c;
uint256 b;
```

这些变量是可以被打包的。因为`c`和`a`打包之后不会超过32字节，他们可以被存放在一个插槽中。

在选择数据类型时，留心变量打包，如果刚好可以与其他变量打包放入一个储存插槽中，那么使用一个小数据类型是不错的。如果`uint128`不能被打包，那么选择`uint256`

**数据位置**

变量打包只发生在存储中，内存或者调用数据是不会打包的。打包函数参数或者本地变量对节省空间是没有帮助的。

**引用数据类型**

结构和数组经常会被放在一个新的储存插槽中。但是他们的内部数据是可以正常打包的。一个`uint8`数组会比`uint256`数组占用更小的空间。

在初始化结构时，分开赋值比一次性赋值会更有效。分开赋值使得优化器一次性更新所有变量。

初始化结构如下：

```
Point storage p = Point()
p.x = 0;
p.y = 0;
```

而非如下：

```
Point storage p = Point(0, 0);
```


**继承**

当你扩展一个合约时，在子合约中的变量可以同父合约中的变量一起打包。

变量的顺序是由[C3 linearization](https://en.wikipedia.org/wiki/C3_linearization)决定的。大部分的情况下，你只要知道子合约变量都在父合约变量之后。

## 数据类型
在选择数据类型以优化Gas时，我们必须权衡利弊。相同的数据类型在不同的情况会也会有便宜或昂贵之分。

**内存和存储**

在内存中进行运行或者调用数据（同内存中运行一样），都是比存储便宜的。

减少存储操作的一种常见方法是在分配给存储变量之前，对本地内存变量其进行操作。

我们经常看到这样的循环：

```
uint256 return = 5; // assume 2 decimal places
uint256 totalReturn;
function updateTotalReturn(uint256 timesteps) external {
    uint256 r = totalReturn || 1;
    
    for (uint256 i = 0; i < timesteps; i++) {
        r = r * return;
    }
    totalReturn = r;
}
```

在`calculateReturn`函数中，我们使用本地内存变量`r`用来存放中间变量，在最后将给过赋值给`totalReturn`。

**固定和动态**

固定大小的数组变量一般比变长数组变量便宜

如果我们知道一个数组有多少元素，我们优先采用固定大小的方式：

```
uint256[12] monthlyTransfers;
```

同样的道理也适用于字符型，一个`string`或者`bytes`变量是变长的。如果一个字符很短，我们可以使用`byte32`

如果我们必须需要一个动态数组，最好将函数设计成加，而不是减的。消耗固定的Gas，而截断数组消耗Gas线性增长。

**映射和数组**

大多数的情况下，使用映射会优于数组。

但是，如果是使用较小的数据类型，数组是一个不错的选择。数组元素会像其他存储变量被打包，节省的存储空间可能会弥补更昂贵数组操作。这个方法在处理大型数组时很有用。

## 其他方式

在处理变量时，还有一些其他技术可以帮助我们优化Gas成本。

**初始化**

在Solidity中，每个变量的赋值都要消耗Gas。在初始化变量时，我们经常会设置永远不会使用的默认值。

`uint256 value;` is cheaper than `uint256 value = 0;`.
`uint256 value;`比`uint256 value = 0;`更便宜。

**Require字符串**

如果你在require中增加语句，你可以通过限制字符串长度为32字节来降低Gas消耗。


**不打包变量**

以太坊虚拟机一次处理32字节，变量大小小于32字节的会被转化。如果你打包变量没有节省Gas，那么直接使用`uint256`会更便宜。

**删除**

当我们删除变量时，以太坊会给我们退款。它的目的是为了鼓励节约区块链上的空间，我们用它来减少交易的Gas成本。

删除一个变量可以退15,000起，最高可达交易消耗Gas的一半。使用“delete”关键字进行删除相当于为数据类型分配初始值，比如为整数分配“0”。

**储存在事件中**

那些不需要在链上被访问的数据可以存放在事件中来达到节省Gas的目的。

虽然可以这个操作，但不推荐使用——事件并不是用于数据存储。如果我们需要的数据存储在很久以前发出的事件中，由于需要搜索的块数量太多，获取这个数据可能会非常耗时。


# 优化函数

*Gas Optimization in Solidity Part II: Functions* coming soon…
*接下来在Solidity中如何优化Gas第二部分：函数*

原文链接：https://medium.com/coinmonks/gas-optimization-in-solidity-part-i-variables-9d5775e43dde
作者：[Will Shahda](https://medium.com/@ethdapp)

