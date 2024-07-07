# 前言

在上篇文章里，我们提到，如果要实现协议验证的过程，Prover 在开始的时候通过多项式编码的方式对 $W$ 表格的每一列进行编码，然后将编码后的结果发给 Verifier 去验证；通过进一步的交互，可以验证等式

$$
q_{L}(X) \cdot w_{a}(X) + q_{R}(X) \cdot w_{b}(X) + q_{M}(X)\cdot(w_{a}(X)\cdot w_{b}(X)) + q_{C}(X) -  q_{O}(X)\cdot w_{c}(X) \overset{?}{=} 0
$$

是否成立。

这是仅在 Prover 是诚实的情况下，我们去验证一个电路的正确性。但是 Prover 很有可能会作恶，为了预防 Prover 作恶，保证电路的正确性和安全性，并在满足约束的情况下，我们需要再验证 $(\sigma_a(X),\sigma_b(X),\sigma_c(X))$ 与 $(w_a(X),w_b(X),w_c(X))$ 之间的关系。

其中：

$(\sigma_a(X),\sigma_b(X),\sigma_c(X))$ 是 Prover 计算的多项式，它们通常表示电路中的输入和中间结果；

$(w_a(X),w_b(X),w_c(X))$ 是预先定义的权重多项式，它们表示电路的结构。

简单来说，其实整个验证的过程最终就是把电路的正确性检查转换成一组加法/乘法的多项式约束。

</br>

为什么要这样的做呢？当然是**提高效率，降低成本**。

首先，

我们以第一章里的矩阵 $W$ 的表格形式「表1-1」为例，它遵循纵向编码的原则，也就是说每一列都可以用一个多项式去表示这一列下的内容计算规则。

目前只有三列，所以即便是逐行检查也不需要花太多时间，但是一旦行数增多到 100 行，那么一行行检查就特别耗费精力，Verifier 需要很强的算力。

因此，可以用一个/若干的多项式去编码三列，把 3×100 的表，变成 3 个多项式来表示。（这点可以结合 excel 表格计算公式来理解，很类似）

<img src="/ZKP-PLONK/images/PLONK多项式编程/表1-1.png" width="40%" />
「表1-1」


其次，变成多项式还有一个天生自带的 buff，假设矩阵 $W$ 的表格可以成功地只用多项式来表示，当 Verifier 进行验证的时候，Verifier 并不需要验证所有域上的点，就可以成功验证电路的正确性。

</br>

**具体来说：**

1. 假设电路的约束可以表示成为一个 3×100 的表，每一行代表一个约束条件。
2. 通过多项式编码的方式，将这 3×100 的表压缩成三个多项式 $\sigma_a(X),\sigma_b(X),\sigma_c(X)$。这些多项式可以通过某种方式（例如拉格朗日插值）从原始约束中构造出来。
3. Verifier 只需要检查这三个多项式在某一个随机点 $X=z$ 上是否满足约束条件。具体来说，Verifier 会检查：

$$
\sigma_a(z)⋅\sigma_b(z)=\sigma_c(z)
$$

如果这个等式在随机点 $z$ 上成立，那么可以高度概率地认为 3×100 这个表里的内容在整个域上都是成立的。

我们结合实例再看一遍，看看具体是怎么进行计算的：

<img src="/ZKP-PLONK/images/PLONK多项式编程/表1-1.png" width="40%" />
「表1-1」

</br>

还是以「表1-1」 为例，假设目前表格只有 3×3 ，那么表格中给出的插值点 i 及其对应的值如下：

- 插值点：i = 1,2,3
- 对应的值：
    - $w_a$: $x_5,x_1,x_3$
    - $w_b$: $x_6,x_2,x_4$
    - $w_c$: $out,x_5,x_6$

</br>

接下来我们**开始构造拉格朗日基函数**：

对于每个 i，拉格朗日基函数 ${l_i}{(X)}$ 的定义是：

$$
{l_i}{(X)}=\prod_{j\neq i}^{} \frac{X-j}{i-j} 
$$

具体到插值点：i = 1,2,3，我们分别求出 ${l_1}{(X)},{l_2}{(X)},{l_3}{(X)}$，

- 对于 i = 1，

$$
{l_1}{(X)}=\frac{(X-2)(X-3)}{(1-2)(1-3)} =\frac{(X-2)(X-3)}{2} 
$$

- 对于 i = 2，

$$
{l_2}{(X)}=\frac{(X-1)(X-3)}{(2-1)(2-3)} = -(X-1)(X-3)
$$

- 对于 i = 3，

$$
{l_3}{(X)}=\frac{(X-1)(X-2)}{(3-1)(3-2)} =\frac{(X-1)(X-2)}{2} 
$$

接下来，结合「表1-1」 中的内容，和上面已经求出的对应 i 的拉格朗日基函数，**我们开始构造插值多项式**：

对于 $w_a$

$$
\sigma_a(X)=x_5 \cdot l_1(X)+x_1 \cdot l_2(X) +x_3 \cdot l_3(X)
$$

把 ${l_1}(X)$ 代入，

$$
\begin{align}
\sigma_a(X) & = x_5 \cdot l_1(X)+x_1 \cdot l_2(X) +x_3 \cdot l_3(X) & \\ & = x_5 \cdot \frac{(X-2)(X-3)}{2} - x_1 \cdot(X-1)(X-3)+x_3 \cdot\frac{(X-1)(X-2)}{2}
\end{align}
$$

对于 $w_b$

$$
\sigma_b(X)=x_6 \cdot l_1(X)+x_2 \cdot l_2(X) +x_4 \cdot l_3(X)
$$

把 ${l_2}(X)$ 代入，

$$
\begin{align}
\sigma_b(X) & = x_6 \cdot l_1(X)+x_2 \cdot l_2(X) +x_4 \cdot l_3(X)\\ & = x_6 \cdot \frac{(X-2)(X-3)}{2} - x_2 \cdot (X-1)(X-3) +x_4 \cdot \frac{(X-1)(X-2)}{2}
\end{align}
$$

对于 $w_c$

$$
\sigma_c(X)=out \cdot l_1(X)+x_5 \cdot l_2(X) +x_6 \cdot l_3(X)
$$

把 ${l_3}(X)$ 代入，

$$
\begin{align}
\sigma_c(X) & = out \cdot l_1(X)+x_5 \cdot l_2(X) +x_6 \cdot l_3(X)\\ & = out \cdot \frac{(X-2)(X-3)}{2} - x_5 \cdot (X-1)(X-3) +x_6 \cdot \frac{(X-1)(X-2)}{2}
\end{align}
$$

构造插值多项式结束以后，如果你想检查所得的 $\sigma_c(X)$ 是否对应表内的值，我们可以代入 X 的值进行验证，比如：

- 当 X=1 的时候，看 $\sigma_c(1)$ 是否 = out ，计算过程如下：

$$
\begin{align}
\sigma_c(1) & = out \cdot \frac{(1-2)(1-3)}{2} - x_5 \cdot (1-1)(1-3) +x_6 \cdot \frac{(1-1)(1-2)}{2} \\ & = out
\end{align}
$$

- 当 X=2 的时候，看 $\sigma_c(2)$ 是否 = $x_5$ ，计算过程如下：

$$
\begin{align}
\sigma_c(2) & = out \cdot \frac{(2-2)(2-3)}{2} - x_5 \cdot (2-1)(2-3) +x_6 \cdot \frac{(2-1)(2-2)}{2} \\ & = x_5
\end{align}
$$

- 当 X=3 的时候，看 $\sigma_c(3)$ 是否 = $x_6$ ，计算过程如下：

$$
\begin{align}
\sigma_c(3) & = out \cdot \frac{(3-2)(3-3)}{2} - x_5 \cdot (3-1)(3-3) +x_6 \cdot \frac{(3-1)(3-2)}{2} \\ & = x_6
\end{align}
$$

同样可以验证 $\sigma_a(X)$ 和 $\sigma_b(X)$。

如果验证通过，那么非常好，说明我们的式子计算没有问题，前置工作准备结束。

**总结一下**，上面那么多内容其实就是做了这样一项工作：根据拉格朗日插值法，通过「表1-1」里已知的关系构造出「表2」

<img src="/ZKP-PLONK/images/PLONK多项式编程/表1-1.png" width="40%" />
「表1-1」

<img src="/ZKP-PLONK/images/PLONK多项式编程/表1-2.png" width="40%" />
「表1-2」

<img src="/ZKP-PLONK/images/PLONK多项式编程/表2.png" width="70%" />
「表2」

其中，在「表1-1」中，定义域是 $i\in {1,2,3}$，看「表1-2」会更加清晰；而在 $\sigma_a(X)$ 的插值多项式中，定义域就会变得更大，插值多项式的构建其实也间接完成了扩域（后面会详细讲述），所以我们可以代入不仅限于 「表2」中的 X={1,2,3} 的示例。

我们通过已有的三个插值点构造了「表2」 的拉格朗日插值多项式，有了它以后，我们可以带入未知点进行插值点的数值运算，验证是否满足 $\sigma_a(X)\cdot \sigma_b(X)=\sigma_c(X)$。如果随机挑选的 X 可以满足 $\sigma_a(X)\cdot \sigma_b(X)=\sigma_c(X)$，那么说明这个范围内的值都都适用，Prover 没有作恶。


</br>


# 多项式的概率检查
如果你看懂了前言部分，那么接下来的内容就不难理解了。

在许多密码学协议和复杂计算的验证过程中，电路（可以是布尔电路或代数电路）常用于描述和实现计算逻辑。验证这些计算的正确性是一个重要问题，而直接验证每一步计算通常是不切实际的。Schwartz-Zippel 定理提供了一种高效的概率验证方法。

**什么是 Schwartz-Zippel 定理呢？**

通过引入随机挑战值，我们可以将原本需要逐一验证的多个条件合并为一个简化的验证步骤。这种方法利用了「多项式随机挑战」的理论，即通过验证多项式在一个随机点上的值，可以高概率确定两个多项式在整个定义域上的相等性。

具体来说，如果有两个多项式 $f(X)$ 和 $g(X)$，它们的次数均不超过 $d$，那么 Verifier 只需要给出一个随机挑战值 $\zeta \in \mathbb{F}$，计算 $f(\zeta)$ 是否等于 $g(\zeta)$ 即可大概率得知 $f(X) = g(X)$，其中出错的概率 $\leq \frac{d}{|\mathbb{F}|}$。只要保证 $\mathbb{F}$ 足够大，那么检查出错的概率就可以忽略不计。

这个原理就被称为 Schwartz-Zippel 定理。

假如要验证两个向量 $\vec{a} + \vec{b}$ 是否等于 $\vec{c}$，为了可以一步挑战验证，我们要先把三个向量编码成多项式。

一种最直接的方案是把向量当作多项式的「系数」进行编码：

$$
\begin{split}
a(X) &= a_0 + a_1X+a_2X^2 + \cdots + a_{n-1}X^{n-1}\\
b(X) &= b_0 + b_1X+b_2X^2 + \cdots + b_{n-1}X^{n-1}\\
c(X) &= c_0 + c_1X+c_2X^2 + \cdots + c_{n-1}X^{n-1}
\end{split}
$$

显然，如果 $a_i+ b_i = c_i$，那么 $a(X)+b(X)=c(X)$。然后我们可以通过挑战一个随机数 $\zeta$ 来检验三个多项式在 $X=\zeta$ 处的取值，验证：

$$
a(\zeta)+b(\zeta)\overset{?}{=}c(\zeta)
$$

如果上式成立，那么 $\vec{a} + \vec{b}=\vec{c}$ 。


</br>

# Lagrange 插值 与 Evaluation Form
但是，假如我们要验证 $\vec{a}\circ\vec{b}\overset{?}{=}\vec{c}$，用系数编码的方式就不容易处理了。当 $a(X)\cdot b(X)$ 会产生很多的交叉项，这些交叉项的系数来自于 $a(X)$ 和 $b(X)$ 的各个不同次幂的项。

我们可以具体来算一下，假设：

$a(X)=a_0 + a_1X+a_2X^2$

$b(X)=b_0 + b_1X+b_2X^2$

$c(X)=c_0 +c_1X+c_2X^2+c_3X^3+C_4X^4$

那么

$$
\begin{align}
a(X)\cdot b(X) & = (a_0 + a_1X+a_2X^2)\cdot(b_0 + b_1X+b_2X^2) & \\ & = a_0b_0+(a_0b1+b_0a1)X+(a_0b_2+a_1b_1+a_2b_0)X^2+\cdots
\end{align}
$$

可以观察上面的等式，$a_i\cdot b_i$ 和 $c_i$ 的项并不对应到 $X^i$ 的系数，比如 $a_1\cdot b_1$ 的系数出现在 $X^2$ 上，但同时 $X^2$ 项的系数组成还有 $a_0b_2$ 和 $a_2b_0$。而 $c_1$ 是 $X^1$ 的系数。

因此我们需要另一种多项式编码方案，利用拉格朗日基(Lagrange Basis)。如果我们要构造多项式 $a(X)$，使得它在定义域 $H=(w_0, w_1, \ldots w_{N-1})$ 上的取值为 $\vec{a}$，即

$$
\begin{split}
a(w_0) &= a_0 \\
a(w_1) &= a_1 \\
&\vdots \\
a(w_{N-1}) &= a_{N-1} \\
\end{split}
$$


构造插值多项式需要用到拉格朗日基函数： $\{L_i(X)\}_{i\in[0,N-1]}$ 

其中 $L_i(w_i)=1$，

并且对于 $j\neq i$， $L_i(w_j)=0$ ，

然后插值多项式 $a(X$）可以表示为基函数 $L_i(X)$ 和对应取值向量 $\vec{a}$ 的线性组合：

$$
a(X)=a_0\cdot L_0(X) + a_1\cdot L_1(X)+ a_2\cdot L_2(X) + \cdots + a_{N-1}\cdot L_{N-1}(X)
$$

**举个更具体的例子**，假设我们要构造插值多项式 $a(X)$，使得它在定义域 $H'={1,2,3}$ 上的取值为 $\vec{a}=(a_0,a_1,a_2)=(4,5,6)$

相当于：我们想构造一个多项式 $a(X)$，使得 $a(1)=4$, $a(2)=5$, $a(3)=6$；也就是：我们有三个已知插值点 (1,4),(2,5),(3,6)，由此构建插值多项式 $a(X)$。

拉格朗日插值多项式 $a(X)$ 可以表示为：

$a(X)={\textstyle \sum_{i=0}^{N-1}} a_iL_i(X)$

**1. 接下来我们计算拉格朗日基函数**：

对于 i =0:
$L_0(X)=\frac{(X-2)(X-3)}{(1-2)(1-3)}=\frac{(X-2)(X-3)}{2}$

对于 i=1:
$L_1(X)=\frac{(X-1)(X-3)}{(2-1)(2-3)}=-(X-1)(X-3)$

对于 i=2:
$L_2(X)=\frac{(X-1)(X-2)}{(3-1)(3-2)}=\frac{(X-1)(X-2)}{2}$

**2. 构造插值多项式 a(X)**:

$$
\begin{align}
a(X) & = 4L_0(X)+5L_1(X)+6L_2(X) & \\ 
& = 4\cdot \frac{(X-2)(X-3)}{2}-5\cdot (X-1)(X_3)+6\cdot \frac{(X-1)(X-2)}{2} \\
& = 2(X-2)(X-3)-5(X-1)(X-3)+3(X-1)(X-2)\\
& = X+3
\end{align}
$$

**3. 验证多项式**

$$
\begin{align}
a(1)=1+3=4\\
a(2)=2+3=5\\
a(3)=3+3=6
\end{align}
$$

验证结果与取值向量 $\vec{a}=(4,5,6)$ 一致。我们构造了多项式 $a(X)$ 并验证其在插值节点处的取值是正确的。


看起来 $L_i(X)$ 像是一个选择器，这意味着：当 $X=w_i$ 时，只有 $L_i(X)$ 为 1，其他所有的 $L_j(X)(j\neq i)$ 都为 0，这是拉格朗日基函数的选择性性质。因此，拉格朗日基函数 $L_i(X)$ 在 $X=w_i$ 时「选择」了对应的系数 $a_i$，而忽略了其他系数，具体看下式：

$$
\begin{align}
a(w_i) & = a_i\cdot L_i(w_i)+ {\textstyle \sum_{j\neq i}^{0\le j<N}} a_j\cdot L_j(w_i)
\end{align}
$$

由于 $L_i(w_i)=1$ ，且对于 $j\neq i$， $L_j(w_i)=0$，

所以 $a(w_i)=a_i\cdot 1 + {\textstyle \sum_{j\neq i}^{0\le j<N}} a_j\cdot 0 =a_i$

我们用同样的方法来编码 $b(X)$ 和 $c(X)$：

$$
\begin{split}
b(X)=b_0\cdot L_0(X) + b_1\cdot L_1(X)+ b_2\cdot L_2(X) + \cdots + b_{N-1}\cdot L_{N-1}(X) \\
c(X)=c_0\cdot L_0(X) + c_1\cdot L_1(X)+ c_2\cdot L_2(X) + \cdots + c_{N-1}\cdot L_{N-1}(X) \\
\end{split}
$$

同理也可得出：

$$
\begin{align}
b(w_i)=b_i\\
c(w_i)=c_i
\end{align}
$$

如果 $a_i\cdot b_i = c_i$ 成立，那么 $a(w_i)\cdot b(w_i) = c(w_i)$。如果 $\vec{a}\circ\vec{b}{=}\vec{c}$ ，那么

$$
a(X)\cdot b(X) = c(X),\quad \forall X\in H
$$

具体的运算：

$$
\begin{split}
a(X)\cdot b(X) = ( {\textstyle \sum_{i = 0}^{N-1}} a_i\cdot l_i(X))\cdot( {\textstyle \sum_{i = 0}^{N-1}b_i\cdot L_i(X}) )
\\ = {\textstyle \sum_{i = 0}^{N-1}}{\textstyle \sum_{i = 0}^{N-1}}(a_i\cdot b_j)\cdot L_i(X)\cdot L_j(X)
\end{split}
$$

因为拉格朗日基函数的选择性性质：
当 $k=i$， $L_i(w_k)=1$；
当 $k\neq i$， $L_i(w_k)=0$ 。

代入 $X=w_k$:

$$
a(w_k)\cdot b(w_k)=( {\textstyle \sum_{i=0}^{N-1}} a_i \cdot L_i(w_k))\cdot ( {\textstyle \sum_{i=0}^{N-1}} b_i\cdot L_i(w_k))
$$

所以 $a(w_k)=a_k$ ， $b(w_k)=b_k$

所以  $a(w_k)\cdot b(w_k)=a_k\cdot b_k =c_k=c(w_k)$

我们现在已经把两个向量的按位乘积问题转换到了三个多项式之间的关系，接下来的问题是如何进行随机挑战验证。

我们发现：如果直接让 Verifier 发送随机数 $\zeta$ 挑战上面的等式，那么 $\zeta$ 只能属于 $H$。如果只存在一个 $j$ 使得 $a_j\cdot b_j\neq c_j$，那么 Verifier 的一次挑战能发现这个错误的概率只有 $\frac{1}{|n|}$，这样 Verifier 需要挑战多次才能缩小检测出错的概率。不过这样不满足我们的要求，我们希望只通过一次挑战来检测出 Prover 的作弊行为。

我们可以把上面的等式的 $X$ 取值范围去除，换成下面的等式：

$$
a(X)\cdot b(X) - c(X) = q(X)\cdot z_H(X), \quad\forall X\in \mathbb{F}
$$

这个等式在整个 $\mathbb{F}$ 定义域上都成立。这是为何？

首先我们看等式左边的多项式： $a(X)\cdot b(X)-c(X)$，不妨定义为 $f(X)$。

我们可以看到 $f(X)$ 在 $X\in H$ 上等于零，那么意味着 $H$ 恰好是 $f(X)$ 的「根集合」， $f(X)$ 可以被表示为零化多项式 $z_H(X)$ 和某个商多项式 $q(X)$ 的乘积，我们可以利用 $z_H(X)$ 的已知性质来简化问题的处理。于是 $f(X)$ 可以按照下面的方式进行因式分解：

$$
f(X)=(X-w_0)(X-w_1)(X-w_2)\cdots(X-w_{N-1})\cdot q(X)
$$

换个说法， $f(X)$ 可以被多项式 $z_H(X)=(X-w_0)(X-w_1)(X-w_2)\cdots(X-w_{n-1})$ 整除，并得到一个商多项式 $q(X)$，它表示 $f(X)$ 除以 $z_H(X)$ 后得到的结果。零多项式 $z_H(X)$ 又被称为消失多项式(Vanishing Polynomial)，它捕捉了 $f(X)$ 的所有根。

补充：
1.右边的 $z_H(X)$ 是一个节点多项式，表示的是这个多项式 $z_H(X)$ 在所有的插值节点 w_i 处都有根，即一个多项式的函数值在所有节点 $w_i$ 处都为零，也就是对于所有 $w_i\in H$， $Z_H(w_i)=0$。这个多项式可以被定义为： $z_H(X)=(X-w_0)(X-w_1)(X-w_2)\cdots(X-w_{n-1})$
2. 多项式的整除定义：假设有两个多项式 $A(X)$ 和 $B(X)$，如果存在一个多项式 $Q(X)$，使得 $A(X)=B(X)\cdot Q(X)$ ，那么我们就说多项式 $A(X)$ 被 $B(X)$ 整除，或者说 $B(X)$ 是 $A(X)$ 的因子。


如果我们让 Prover 计算出这个 $q(X)$，并且发送给 Verifier，又因为 $H$ 是已知的系统参数，Verifier 可以自行计算 $z_H(X)$，那么 Verifier 只需要一次随机检测即可判断 $a(X)\cdot b(X)-c(X)$ 是否在 $H$ 处等零。

$$
a(\zeta)\cdot b(\zeta)-c(\zeta) \overset{?}{=} q(\zeta)\cdot z_H(\zeta)
$$

进一步来说，如果我们使用多项式承诺（Polynomial Commitment），Verifier 可以让 Prover 来帮忙计算这些多项式在 $X=\zeta$ 处的取值，并生成一个证明，证明取值的正确性难过，Verifier 只需要检查这个证明的有效性，而不需要自己计算多项式的取值，这样能最大限度地减少 Verifier 的工作量。

但是， Verifier 计算 $z_H(\zeta)$ 需要 $O(n)$ 的计算量，因为要进行 n 次乘法操作。

补充：在计算复杂度中，$O(n)$ 表示随着输入规模 $n$ 的增加，算法的运行时间或所需的资源（如计算步骤）会以线性比例增加。简而言之，如果一个算法是 $O(n)$ 的，那么当输入数据量增加一倍时，算法的运行时间也会增加一倍。

那能否让 Verifier 继续减少工作量？答案是可以的，只要我们选择特殊的 $H\subset \mathbb{F}$ ，具体来说，如果选择 $H$ 为某种结构良好的集合，可以利用快速傅里叶变换（FFT）等算法来高效计算零多项式 $z_H(X)$ 和其取值。



</br>


## 单位根 Roots of Unity

如果我们选择单位根作为 $H$，那么 $z_H(\zeta)$ 的计算量会降为 $O(\log{n})$。

对于任何有限域 $\mathbb{F}_p=(0,1,\ldots,p-1)$，其中阶数 $p$ 为素数。那么去除零之后剩下的元素构成了乘法群 $\mathbb{F}_p^\ast=(1,\ldots,p-1)$，阶数为 $p-1$。由于 $p-1$ 一定为偶数，那么 $p-1$ 的乘法因子中一定包含若干个 $2$，假设记为 $\lambda$ 个 $2$。那么 $\mathbb{F}_p^\ast$ 一定包含一个阶数为 $2^\lambda$ 的乘法子群。不妨设 $n=2^{k}, k\leq\lambda$，那么一定存在一个阶数为 $n$ 的乘法子群，记为 $H$。 该乘法子群必然含有一个生成元，记为 $\omega$，并且 $\omega^N=1$。这相当于把 $1$ 开 $N$ 次方根，因此被称为单位根。不过单位根不只有一个 $\omega$，我们会发现 $\omega^2,\omega^3,\ldots,\omega^{N-1}$ 都满足单位根的特性，即 $(\omega^k)^N=1, k\in(2,3,\ldots,N-1)$。那么所有这些由 $\omega$ 产生的单位根就组成了乘法子群 $H$：

$$
H=(1,\omega,\omega^2,\omega^3,\ldots,\omega^{N-1})
$$

这些元素满足一定的对称性：比如 $\omega^{\frac{N}{2}}=-1$ ， $\omega=-\omega^{\frac{N}{2}+1}$，
 $\omega^i=-\omega^{\frac{N}{2}+i}$。又比如把所有的单位根求和，我们会得到零：

$$
\sum_{i=0}^{N-1}\omega^i=0
$$

举一个简单的例子，我们可以在 $\mathbb{F}_{13}$ 中找到一个阶数为 $4$ 的 $H$。 

$$
\mathbb{F}_{13}=(0,1,2,3,4,5,6,7,8,9,10,11,12)
$$

其中乘法群的生成元为 $g=2$。由于 $13-1=3\*2\*2$，所以存在一个阶数为 $4$ 的乘法子群，其生成元为 $\omega=5$：

$$
H=(\omega^0=1,\omega^1=5,\omega^2=12,\omega^3=8)
$$

而 $\omega^4=1=\omega^0$。

在实际应用中，我们会选择一个较大的有限域，它能有一个较大的 Powers-of-2 乘法子群。比如椭圆曲线 `BN254` 的 Scalar Field，含有一个阶数为 $2^{28}$ 的乘法子群，`BLS-12-381` 的Scalar Field 含有一个阶数为 $2^{32}$ 的乘法子群。

在乘法子群 $H$ 上，具有下面的性质：

$$
z_H(X)=\prod_{i=0}^{N-1}(X-\omega^i)=X^N-1
$$

我们可以进行简单的推导，假设 $N = 4$，由于 $\omega^i$ 的对称性，这个计算过程可以不断化简：

$$
\begin{split}
&(X-\omega^0)(X-\omega^1)(X-\omega^2)(X-\omega^3) \\
=& (X-1)(X-\omega)(X+1)(X-\omega^{3})  \\
=& (X^2-1)(X-\omega)(X+\omega) \\
=& (X^2-1)(X^2-\omega^2) \\
=& (X^2-1)(X^2+1) \\
=& (X^4-1) \\
\end{split}
$$


## Lagrange Basis

对于 Lagrange 多项式， $L_i(w_i)=1$，并且 $L_i(w_j)=0, (j\neq i)$。接下来，我们给出 $L_i(X)$ 的构造。

为了构造 $L_i(X)$，先构造不等于零的多项式部分。由于 $L_i(\omega_j)=1, j = i$，因此他一定包含 $\prod_{j,j\neq i}(X-\omega_j)$ 这个多项式因子。但该因子显然在 $X=\omega_i$ 处可能不等于 $1$，即可能 $\prod_{j, j\neq i}(\omega_i-\omega_j)\neq 1$。然后，我们只要让该因子除以这个可能不等于 $1$ 的值即可，于是 $L_i(X)$ 定义如下：

$$
L_i(X) = \frac{\prod_{j\in H\backslash\{i\}}(X-\omega_j)}{\prod_{j\in H\backslash\{i\}}(\omega_i-\omega_j)} = \prod_{j\in H\backslash\{i\}}^{} \frac{X-\omega_j}{\omega_i-\omega_j}
$$

不难发现， $L_i(X)$ 在 $X=\omega_i$ 处等于 $1$，其它位置 $X=\omega_j, j\neq i$ 处等于 $0$。

对于任意次数小于 $N$ 的多项式 $f(X)$，那么它都可以唯一地表示为：

$$
f(X)=a_0\cdot L_0(X)+a_1\cdot L_1(X)+a_2\cdot L_2(X)+ \cdots + a_{N-1}\cdot L_{N-1}(X)
$$

我们可以用多项式在 $H$ 上的值 $(a_0,a_1,a_2,\ldots,a_{N-1})$ 来表示 $f(X)$。这被称为多项式的求值形式（Evaluation Form），区别于系数形式（Coefficient Form）。

两种形式可以在 $H$ 上可以通过 (Inverse) Fast Fourier Transform 算法来回转换，计算复杂度为 $O(N\log{N})$。


</br>

## 多项式的约束

利用 Lagrange Basis 我们可以方便地对各种向量计算进行约束。

比如我们想约束 $\vec{a}=(h,a_1,a_2,\ldots,a_{N-1})$ 向量的第一个元素为 $h$。那么我们可以对这个向量进行编码，得到 $a(X)$，并且进行如下约束：

$$
L_0(X)(a(X)-h) = 0, \quad \forall X\in H
$$

Verifier 可以挑战验证下面的多项式等式：

$$
L_0(X)(a(X)-h) = q(X)\cdot z_H(X)
$$

再比如，我们想约束 $\vec{a}=(h_1,a_1,a_2,\ldots,a_{N-2},h_2)$ 向量的第一个元素为 $h_1$，最后一个元素为 $h_2$，其它元素任意。那么 $a(X)$ 应该满足下面两个约束。

$$
\begin{split}
L_0(X)\cdot (a(X)-h_1) &= 0, \quad \forall X\in H\\
L_{N-1}(X)\cdot(a(X)-h_2) &= 0, \quad \forall X\in H
\end{split}
$$

那么通过 Verifier 给一个随机挑战数（ $\alpha$），上面两个约束可以合并为一个多项式约束：

$$
L_0(X)\cdot (a(X)-h_1) + \alpha\cdot L_{n-1}(X)\cdot(a(X)-h_2) = 0, \quad \forall X\in H
$$

接下来，Verifier 只要挑战下面的多项式等式即可：

$$
L_0(X)\cdot (a(X)-h_1) + \alpha\cdot L_{n-1}(X)\cdot(a(X)-h_2) = q(X)\cdot z_H(X)
$$

如果想验证 $\vec{a}$ 和 $\vec{b}$ 两个等长向量除第一个元素之外，其它元素都相等，那要如何约束呢？假设 $a(X)$ 和 $b(X)$ 为两个向量的多项式编码，那么它们应该满足：

$$
(X-\omega^0)(a(X)-b(X))=0
$$

当 $X=\omega^0$ 时，左边多项式的第一个因子等于零，而 $X\in H\backslash\\{\omega^0\\}$ 时，则左边第二因子等于零，即表达了除第一项可以不等之外，其它点取值都必须相等。

可以看出，采用 Lagrange 多项式，我们可以灵活地约束多个向量之间的关系，并且可以把多个约束合并在一起，让 Verifier 仅通过很少的随机挑战就可验证多个向量约束。

## Coset

在素数有限域的乘法群中，对于每一个乘法子群 $H$，都有多个等长的陪集（Coset），这些 Coset 具有和 $H$ 类似的性质，在 Plonk 中也会用到 Coset 的概念，这里只做部分性质的介绍。

还拿 $\mathbb{F}_{13}$ 为例，我们取 $H=(1,5,12,8)$，并且乘法群的生成元 $g=2$。于是我们可以得到下面两个 Coset：

$$
\begin{split}
H_1 &= g\cdot H  = (g, g\omega, g\omega^2, g\omega^3) &= (2,10,11,3) \\
H_2 &= g^2\cdot H = (g^2, g^2\omega, g^2\omega^2, g^2\omega^3) &= (4,7,9,6) \\
\end{split}
$$

可以看到 $\mathbb{F}^*_{13}=H\cup H_1 \cup H_2$，并且它们交集为空，没有任何重叠。并且它们的 Vanishing Polynomial 也可以快速计算：

$$
z_{H_1}(X)=X^N-g^N, \quad z_{H_2}(X)=X^N-g^{2N}
$$


## References

- Schwartz–Zippel lemma. https://en.wikipedia.org/wiki/Schwartz%E2%80%93Zippel_lemma
