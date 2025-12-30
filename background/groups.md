# Cryptographic groups

In the section [Inverses and groups](fields.md#inverses-and-groups) we introduced the
concept of *groups*. A group has an identity and a group operation. In this section we
will write groups additively, i.e. the identity is $\mathcal{O}$ and the group operation
is $+$.

Some groups can be used as *cryptographic groups*. At the risk of oversimplifying, this
means that the problem of finding a discrete logarithm of a group element $P$ to a given
base $G$, i.e. finding $x$ such that $P = [x] G$, is hard in general.

## Pedersen commitment
The Pedersen commitment [[P99]] is a way to commit to a secret message in a verifiable
way. It uses two random public generators $G, H \in \mathbb{G},$ where $\mathbb{G}$ is a
cryptographic group of order $q$. A random secret $r$ is chosen in $\mathbb{Z}_q$, and the
message to commit to $m$ is from any subset of $\mathbb{Z}_q$. The commitment is 

$$c = \text{Commit}(m,r)=[m]G + [r]H.$$ 

To open the commitment, the committer reveals $m$ and $r,$ thus allowing anyone to verify
that $c$ is indeed a commitment to $m.$

[P99]: https://link.springer.com/content/pdf/10.1007%2F3-540-46766-1_9.pdf#page=3

Notice that the Pedersen commitment scheme is homomorphic:

$$
\begin{aligned}
\text{Commit}(m,r) + \text{Commit}(m',r') &= [m]G + [r]H + [m']G + [r']H \\
&= [m + m']G + [r + r']H \\
&= \text{Commit}(m + m',r + r').
\end{aligned}
$$

Assuming the discrete log assumption holds, Pedersen commitments are also perfectly hiding
and computationally binding:

* **hiding**: the adversary chooses messages $m_0, m_1.$ The committer commits to one of
  these messages $c = \text{Commit}(m_b,r), b \in \{0,1\}.$ Given $c,$ the probability of
  the adversary guessing the correct $b$ is no more than $\frac{1}{2}$.
* **binding**: the adversary cannot pick two different messages $m_0 \neq m_1,$ and
  randomness $r_0, r_1,$ such that $\text{Commit}(m_0,r_0) = \text{Commit}(m_1,r_1).$

### Vector Pedersen commitment
We can use a variant of the Pedersen commitment scheme to commit to multiple messages at
once, $\mathbf{m} = (m_0, \cdots, m_{n-1})$. This time, we'll have to sample a corresponding
number of random public generators $\mathbf{G} = (G_0, \cdots, G_{n-1}),$ along with a
single random generator $H$ as before (for use in hiding). Then, our commitment scheme is:

$$
\begin{aligned}
\text{Commit}(\mathbf{m}; r) &= \text{Commit}((m_0, \cdots, m_{n-1}); r) \\
&= [r]H + [m_0]G_0 + \cdots + [m_{n-1}]G_{n-1} \\
&= [r]H + \sum_{i= 0}^{n-1} [m_i]G_i.
\end{aligned}
$$

> TODO: is this positionally binding?

## Diffie–Hellman

An example of a protocol that uses cryptographic groups is Diffie–Hellman key agreement
[[DH1976]]. The Diffie–Hellman protocol is a method for two users, Alice and Bob, to
generate a shared private key. It proceeds as follows:

1. Alice and Bob publicly agree on two prime numbers, $p$ and $G,$ where $p$ is large and
   $G$ is a primitive root $\pmod p.$ (Note that $g$ is a generator of the group
   $\mathbb{F}_p^\times.$)
2. Alice chooses a large random number $a$ as her private key. She computes her public key
   $A = [a]G \pmod p,$ and sends $A$ to Bob.
3. Similarly, Bob chooses a large random number $b$ as his private key. He computes his
   public key $B = [b]G \pmod p,$ and sends $B$ to Alice.
4. Now both Alice and Bob compute their shared key $K = [ab]G \pmod p,$ which Alice
   computes as
   $$K = [a]B \pmod p = [a]([b]G) \pmod p,$$
   and Bob computes as
   $$K = [b]A \pmod p = [b]([a]G) \pmod p.$$

[DH1976]: https://ee.stanford.edu/~hellman/publications/24.pdf

A potential eavesdropper would need to derive $K = [ab]g \pmod p$ knowing only
$g, p, A = [a]G,$ and $B = [b]G$: in other words, they would need to either get the
discrete logarithm $a$ from $A = [a]G$ or $b$ from $B = [b]G,$ which we assume to be
computationally infeasible in $\mathbb{F}_p^\times.$

More generally, protocols that use similar ideas to Diffie–Hellman are used throughout
cryptography. One way of instantiating a cryptographic group is as an
[elliptic curve](curves.md). Before we go into detail on elliptic curves, we'll describe
some algorithms that can be used for any group.

## Multiscalar multiplication

### TODO: Pippenger's algorithm
Reference: https://jbootle.github.io/Misc/pippenger.pdf


# 密码学群

在[逆元与群](fields.md#inverses-and-groups)章节中，我们介绍了*群*的概念。一个群具有单位元和群运算。本节我们将使用加法符号表示群运算，即单位元为$\mathcal{O}$，群运算符为$+$。

## Pedersen承诺

Pedersen承诺[[P99]]是一种可验证的秘密消息承诺方案。该方案使用两个随机公开生成元$G, H \in \mathbb{G}$（其中$\mathbb{G}$是阶为$q$的密码学群）。随机选择秘密值$r \in \mathbb{Z}_q$，待承诺消息$m \subseteq \mathbb{Z}_q$，其承诺值为：

$$c = \text{Commit}(m,r)=[m]G + [r]H$$

验证时，承诺方公开$m$和$r$，任何人都可验证$c$是否为$m$的有效承诺。

[P99]: https://link.springer.com/content/pdf/10.1007%2F3-540-46766-1_9.pdf#page=3

### 同态性质
Pedersen承诺具有同态性：
$$
\begin{aligned}
\text{Commit}(m,r) + \text{Commit}(m',r') 
&= [m]G + [r]H + [m']G + [r']H \\
&= [m + m']G + [r + r']H \\
&= \text{Commit}(m + m',r + r')
\end{aligned}
$$

### 安全属性
在离散对数假设下，Pedersen承诺满足：
- ​**完美隐藏性**：敌手选择消息$m_0, m_1$，承诺方生成$c = \text{Commit}(m_b,r)$，敌手猜中$b$的概率不超过$\frac{1}{2}$
- ​**计算绑定性**：敌手无法找到$m_0 \neq m_1$和$r_0, r_1$使得$\text{Commit}(m_0,r_0) = \text{Commit}(m_1,r_1)$

### 向量Pedersen承诺
扩展方案支持多消息承诺$\mathbf{m} = (m_0, \cdots, m_{n-1})$，使用生成元组$\mathbf{G} = (G_0, \cdots, G_{n-1})$和$H$：
$$
\begin{aligned}
\text{Commit}(\mathbf{m}; r) 
&= [r]H + \sum_{i=0}^{n-1} [m_i]G_i
\end{aligned}
$$

> 待办事项：是否具有位置绑定性？

## Diffie–Hellman 密钥交换

一个使用密码学群的协议示例是 Diffie–Hellman 密钥交换协议 [[DH1976]]。Diffie–Hellman 协议是两名用户 Alice 和 Bob 生成共享私钥的方法。其流程如下：

1. Alice 和 Bob 公开协商两个素数 $p$ 和 $G$，其中 $p$ 是一个大素数，$G$ 是模 $p$ 的一个本原根。（注意，$G$ 是群 $\mathbb{F}_p^\times$ 的生成元。）
2. Alice 选择一个大的随机数 $a$ 作为她的私钥。她计算她的公钥 $A = [a]G \pmod p$，并将 $A$ 发送给 Bob。
3. 类似地，Bob 选择一个大的随机数 $b$ 作为他的私钥。他计算他的公钥 $B = [b]G \pmod p$，并将 $B$ 发送给 Alice。
4. 现在，Alice 和 Bob 都计算他们的共享密钥 $K = [ab]G \pmod p$，其中 Alice 的计算方式为：
   $$K = [a]B \pmod p = [a]([b]G) \pmod p,$$
   而 Bob 的计算方式为：
   $$K = [b]A \pmod p = [b]([a]G) \pmod p.$$

[DH1976]: https://ee.stanford.edu/~hellman/publications/24.pdf

潜在的窃听者需要仅通过已知的 $g, p, A = [a]G$ 和 $B = [b]G$ 推导出 $K = [ab]G \pmod p$，换句话说，他们需要从 $A = [a]G$ 中获取离散对数 $a$ 或从 $B = [b]G$ 中获取离散对数 $b$，而我们认为这在 $\mathbb{F}_p^\times$ 中是计算上不可行的。

更一般地，使用类似 Diffie–Hellman 思想的协议在密码学中广泛应用。一种实例化密码学群的方法是使用[椭圆曲线](curves.md)。在深入讨论椭圆曲线之前，我们将描述一些适用于任何群的算法。

## 多标量乘法

### TODO: Pippenger 算法
参考：https://jbootle.github.io/Misc/pippenger.pdf
