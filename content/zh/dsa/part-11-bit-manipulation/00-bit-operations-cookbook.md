---
title: "位运算速查手册"
description: "第 11 部分其余内容所依赖的两条恒等式：x & -x 取出最低置位位，x & (x-1) 把它消掉。另外还有 popcount、置位/清零/翻转/测试，以及子集遍历。"
slug: 00-bit-operations-cookbook
part: 11
chapter: 0
difficulty: intermediate
prerequisites:
  - part-0-foundations/03-bit-manipulation-primer
date_updated: 2026-05-25
languages: [python, java, cpp, go]
canonical_test: LC-338
widgets: []
ladder:
  core: [LC-338, LC-1009, LC-201]
  stretch: [LC-89, LC-1521]
  star: LC-338
---

# 位运算速查手册

本章靠两个表达式撑起来：

```text
x & -x         # the value of the lowest set bit
x & (x - 1)    # x with the lowest set bit cleared
```

第 11 部分的其他内容都建立在这两行之上。树状数组（Fenwick tree）用 `i += i & -i` 在索引空间里跳走。Brian Kernighan 的 popcount 靠 `n &= n - 1` 循环。位掩码 DP 用 `sub = (sub - 1) & mask` 遍历子集。单数异或那一族题，靠取出两个数任意一个不同的位把它们分成两组——那个位就是 `(a ^ b) & -(a ^ b)`。把这两条恒等式背下来，第 11 部分剩下的内容就不再像巫术。

其余的几个基本操作（置位、清零、翻转、测试、popcount、遍历置位位）每个只有一行。本章是速查手册，不是叙事文章。先扫一遍。等某道题要用到其中某个操作、而你已经忘了确切写法时，再回来查。

[位运算入门](../part-0-foundations/03-bit-manipulation-primer.md) 讲过 AND、OR、XOR、NOT、移位和 Brian Kernighan 循环。本章是在这些基本操作之上的一层操作词汇，也就是 Hacker's Delight 第 2-1 节收录、JDK 源码注释里用 `// HD, Section 2-1` 标注的那一套。[^1]

## 置位第 i 位

```text
x | (1 << i)
```

`1 << i` 是一个只在第 `i` 位为 1 的掩码。和目标做 OR，就把第 `i` 位强制成 1，其他位一个不动。幂等：对已经置位的位再置一次，什么也不会发生。

```python
def set_bit(x, i):
    return x | (1 << i)
```

## 清零第 i 位

```text
x & ~(1 << i)
```

把单比特掩码取反，就得到除第 `i` 位以外全为 1 的掩码。AND 会把第 `i` 位清零，其余照旧。同样幂等。

Go 为这个组合提供了一个单独的运算符：`x &^ (1 << i)`，读作“x AND NOT (1 << i)”，少写一个符号。

```python
def clear_bit(x, i):
    return x & ~(1 << i)
```

## 翻转第 i 位

```text
x ^ (1 << i)
```

和掩码做 XOR，目标位翻转，其余不动。翻两次回到原值——XOR 是自己的逆运算，这也是 11.1 章全部内容的基础恒等式。

```python
def flip_bit(x, i):
    return x ^ (1 << i)
```

## 测试第 i 位

```text
(x >> i) & 1     # returns 0 or 1
```

把第 `i` 位移到第 0 位，再把其他位全部掩掉。括号不能省。`x >> i & 1` 在 C、C++、Java、Python 里恰好能正确解析（`>>` 的优先级高于 `&`），但代码评审每次都会把它挑出来；而且总有一天你会往同一个表达式里加个 `==`，然后被“`==` 比 `&` 绑得更紧”这件事坑一次。加括号。

```python
def test_bit(x, i):
    return (x >> i) & 1
```

> [!WARNING]
> **输入可能是负数时，Java 要用无符号右移。** Java 的 `>>` 是算术右移，会做符号扩展，所以用 `(x >> i) & 1` 去测一个负 `int` 的高位，永远返回 1。改用 `(x >>> i) & 1`，才能拿到不带符号扩展的位值。C++ 和 Go 的解法是做位运算时改用无符号类型（`uint32_t`、`uint32`）；Python 没有固定位宽，不受影响。

## 取出最低置位位：`x & -x`

两条承重恒等式里的第一条。返回值是一个 2 的幂：`x` 最低置位位的值；`x` 为 0 时返回 0。

证明要靠补码。[^2] 按定义 `-x = ~x + 1`。把 `x` 的每一位取反得到 `~x`，`~x` 末尾那串 1 正好覆盖 `x` 末尾那串 0 所在的位置。给 `~x` 加 1，进位会穿过这串末尾的 1，把它们全部滚回 0，停在 `~x` 的第一个 0 上。这个停下来的位置恰好是 `x` 最低的那个 1，而且它在 `x` 和 `-x` 里都取 1。其余每一位，在这两个操作数中只有一个为 1。AND 只留下共同为 1 的那一位。

拿 `x = 0b01100`（= 12）走一遍：

```text
 x = 01100
~x = 10011
-x = 10100   (~x + 1, the carry hits position 2)
x & -x       = 00100   (only position 2 is set in both)
```

第 2 位是 12 最低的置位位。结果是 `4 = 2^2`，一个 2 的幂：这是那一位的*值*，不是它的下标。

```python
def lowest_set_bit(x):
    return x & -x
```

树状数组在索引上跳走用的就是这个操作：`update(i, v)` 靠 `i += i & -i` 往上走，`query(i)` 靠 `i -= i & -i` 往回退。每次操作 `O(log n)`，因为每一步都消掉一个最低的置位位，而这样的位最多只有 `log2(n)` 个。

## 消去最低置位位：`x & (x - 1)`

第二条承重恒等式。返回把最低置位位去掉之后的 `x`，它上面的每一位都不动。

`x` 减 1 时，借位会穿过 `x` 末尾的 0，把它们翻成 1。借位停在 `x` 最低的置位位上，这一位变成 0。于是 `x - 1` 和 `x` 在最低置位位之上相同，在那一位上不同（`x` 是 1，`x - 1` 是 0），在它之下也不同（`x` 是 0，`x - 1` 是 1）。AND 保留相同的高位，其余清零。

还是 `x = 0b01100`：

```text
x     = 01100
x - 1 = 01011    (borrow rolls through positions 0 and 1)
x & (x - 1) = 01000   (positions 3 above; lowest 1 at position 2 is gone)
```

12 丢掉最低的那个 1，变成 8。

```python
def strip_lowest_set_bit(x):
    return x & (x - 1)
```

分清这两条的口诀：**取位用负 x，消位用 x 减一**。两者看着对称。并不对称。`x & -x` 返回那一位的值；`x & (x - 1)` 返回把那一位删掉后的整数。LC 191 和 LC 338 的提交里，天天有人把两者搞反，要么死循环，要么数错。先把它练成肌肉记忆，再相信它。

推论：`x & (x - 1) == 0` 为真当且仅当 `x` 是 2 的幂（前提 `x > 0`）。2 的幂只有一个置位位，把它消掉就是 0。其他数至少有两个置位位，消掉最低的那个之后仍然非零。

## 统计置位个数：Brian Kernighan 循环

`x & (x - 1)` 的招牌应用。反复消掉最低的置位位，直到 `x` 变成 0，数一数循环了几次。

```python
def popcount(x):
    count = 0
    while x:
        x &= x - 1     # 每轮消掉一个置位位
        count += 1
    return count
```

每一轮恰好清掉一个置位位，从不会置位。循环次数等于置位位的个数，然后退出。`x = 0xFFFFFFFF`（32 位全 1）时跑 32 次，`x = 1` 时跑 1 次。而朴素的 `for k in range(32): count += (x >> k) & 1` 不管输入是什么都跑 32 次。

手工推演 `x = 0b1011`（= 11）：

| 轮次 | 之前的 x | x - 1 | x & (x - 1) | 被清掉的位 | 计数 |
|---|---|---|---|---|---|
| 1 | `1011` | `1010` | `1010` | 第 0 位 | 1 |
| 2 | `1010` | `1001` | `1000` | 第 1 位 | 2 |
| 3 | `1000` | `0111` | `0000` | 第 3 位 | 3 |

循环退出，返回 3。

这个算法最早由 Peter Wegner 发表在 *CACM* 第 3 卷（1960），1964 年 Derrick Lehmer 独立重新发现，后来作为习题 2-9 收进 *The C Programming Language* 第 2 版（1988）——算法因此挂上了 Kernighan 的名字。[^1][^3]

招牌题是 LC 338 Counting Bits。要算 `[0, n]` 里每个整数的 popcount，递推式自然从 `x & (x - 1)` 里掉出来：

```python
def count_bits(n):
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        dp[i] = dp[i & (i - 1)] + 1
    return dp
```

**解法** — Python（[Java](./00-bit-operations-cookbook/lc-338/sol.java) · [C++](./00-bit-operations-cookbook/lc-338/sol.cpp) · [Go](./00-bit-operations-cookbook/lc-338/sol.go)）

```python
# LC 338. Counting Bits


def count_bits(n: int) -> list[int]:
    """Return ans[i] = popcount(i) for i in 0..n.

    Recurrence: dp[i] = dp[i & (i - 1)] + 1.
    Cleared-lowest-bit subproblem already solved; add one for the bit just removed.
    O(n) time, O(n) space (output dominates).
    """
    dp = [0] * (n + 1)
    for i in range(1, n + 1):
        dp[i] = dp[i & (i - 1)] + 1
    return dp
```

子问题 `dp[i & (i - 1)]` 是 `i` 去掉最低位之后的 `popcount`，因为 `i & (i - 1) < i`，它早就算过了。再加 1，补上刚被去掉的那一位。两行代码，`O(n)` 时间。

## 标准库的 popcount：循环是错误答案的时候

生产代码里别手写 Brian Kernighan 循环。每种现代语言都自带 popcount 内建函数（intrinsic），在 x86 SSE4.2 和 ARMv8 上会编译成硬件 `POPCNT` 指令。[^4]

| 语言 | 生产环境里的调用 | 备注 |
|---|---|---|
| Python | `x.bit_count()` (3.10+)；`bin(x).count('1')`（更老的版本） | `int.bit_count` 是在 PEP 632 中加入的。 |
| Java | `Integer.bitCount(x)` / `Long.bitCount(x)` | OpenJDK 实现的是 Hacker's Delight 图 5-2 里的 SWAR 算法，注释写着 `// HD, Section 5-1`。[^1] |
| C++ | `std::popcount(x)`（C++20，`<bit>`）；`__builtin_popcount(x)`（GCC/Clang） | `<bit>` 里还有一批配套内建函数：`std::countl_zero`、`std::countr_zero`、`std::bit_width`、`std::has_single_bit`。只对无符号类型生效。 |
| Go | `bits.OnesCount(x)`（`math/bits`）；带类型的变体 `OnesCount32`、`OnesCount64` | 在 `GOAMD64=v3` 下，或通过 `runtime/internal/cpu` 的特性检查后，编译器会生成 `POPCNT`。 |

循环背后其实藏着三件事。SWAR 算法（*SIMD Within A Register*）用魔数 `0x55555555`、`0x33333333`、`0x0F0F0F0F`，在一个 `w` 位字上以 `O(log w)` 次操作，按半字节、字节、半字并行统计。JDK 的 `Integer.bitCount` 就是这个构造；源码一分钟能读完，是这门技术最干净的生产级参考。[^1] 硬件 `POPCNT` 指令再进一步，无论输入是什么都是固定的三周期延迟。查表法则是预先算好每个 8 位字节的 popcount，32 位字查四次再相加：当你更在意可预测的缓存行为而不是峰值吞吐量时，它才是对的选择。

Brian Kernighan 循环只赢在教学这一轮：它让“答案就是 `popcount(x)`”这个证明变得显然。学的时候用它。上线的时候用内建函数。

## 遍历置位位

题目要求“`x` 的每一个置位位”时，自然的写法是把两条恒等式合起来：

```python
def set_bit_positions(x):
    while x:
        bit = x & -x        # 最低置位位的值
        x &= x - 1          # 消掉它；后面的循环看到的是次低的那个
        yield bit.bit_length() - 1   # 把值换算成下标
```

每一轮：取出最低的置位位（值是一个 2 的幂），对这个下标做题目要求的事，然后把它消掉。循环跑 `popcount(x)` 次。这就是用位掩码遍历集合元素的底层机制：枚举某个掩码的子集时形状相同，只多一个转折（`sub = (sub - 1) & mask`），用来按固定顺序遍历子集，而不只是遍历单个掩码的置位位。[用位掩码枚举子集](../part-7-recursion-backtracking/02-subsets-combinations-permutations.md) 把那处转折变成了承重的那一步。

不想手动把值换算成下标时，标准库里也有直接取最低置位位下标的调用：Java 的 `Integer.numberOfTrailingZeros(x)`、C++20 的 `std::countr_zero(x)`、Go 的 `bits.TrailingZeros(x)`、Python 的 `(x & -x).bit_length() - 1`（没有直接内建，但对大整数友好）。`x = 0` 时四者的返回值都有明确定义：32（Java）、`w`（C++/Go，对宽度为 `w` 的类型），Python 那个表达式则是未定义（返回 `-1`，因为 `(0).bit_length() == 0`）。先做范围检查。

## 运算符优先级与各语言移位陷阱

在 C、C++、Java、Python 里，位运算符的优先级低于比较运算符。`x & 1 == 0` 会被解析成 `x & (1 == 0)` = `x & 0` = `0`，而 0 在每种语言里都是假值。这个条件永远不成立。加括号：`(x & 1) == 0`。GCC 的 `-Wparentheses` 就是为这个反模式加的，每个面试季它都会对着同一个表达式报警。

速查手册的读者还要当心三个跨语言陷阱。

> [!WARNING]
> **C++ 里有符号左移进符号位，是实现定义（C++14+）或未定义行为（C++11）。** 对 32 位有符号 `int` 做 `1 << 31` 不可移植。[^5] 只要真的是位级操作，就用无符号字面量（`1u << 31`）和无符号类型（`uint32_t`、`uint64_t`）。本手册的 C++ 代码全程用 `uint32_t`，就是这个原因。

> [!WARNING]
> **移位量等于或超过操作数位宽时，各语言各走各的。** `1 << 32` 在 C 和 C++ 里是未定义行为。Java 对移位量按操作数位宽取模（所以 `int` 下 `1 << 32 == 1`，`long` 下 `1 << 64 == 1`）。Go 规定结果是 0。Python 是任意精度，直接算出 `4294967296`。代码跨语言复制，含义会悄悄变掉。只要移位量可能来自输入，移位之前先做范围检查。

> [!WARNING]
> **Java 有两个右移运算符，选对那个。** `>>` 是算术右移（符号扩展），`>>>` 是逻辑右移（补零）。做高位有意义的位级操作时用 `>>>`。对负数 `int` 用 `>>` 去提取符号位，拿到的是垃圾值。C++ 和 Go 每种类型只有一个 `>>`，但行为由有符号/无符号决定：有符号 `>>` 做符号扩展，无符号 `>>` 补零。四种语言通用的反射动作是一样的：有得选时，位运算就在无符号类型里做。[^6]

第五个陷阱是同宽掩码构造里的差一错误。LC 1009 Complement of Base 10 Integer 里 `n = 5`（`0b101`），同宽的全 1 掩码是 `0b111 = 7`，不是 `0b1111 = 15`。迭代构造比闭式公式更不容易写错：`mask = 1; while mask < n: mask = (mask << 1) | 1`。循环结束后 `mask >= n`，且 `mask` 是 `0b1...1`，位数和 `n` 一样。然后 `complement = mask ^ n`。

## 练习

本章讲的是速查手册式的基本操作；题目梯度里挑的是那些“手册本身就是解法”的题。更早的 popcount 和汉明距离题（LC 191、LC 461）在[位运算入门](../part-0-foundations/03-bit-manipulation-primer.md)里，这里不重复。

### 核心

- [LC 338 — Counting Bits](https://leetcode.com/problems/counting-bits/) [简单] • bit-dp-on-low-bit ★
- [LC 1009 — Complement of Base 10 Integer](https://leetcode.com/problems/complement-of-base-10-integer/) [简单] • mask-construction *（参考解法：仅 Python）*
- [LC 201 — Bitwise AND of Numbers Range](https://leetcode.com/problems/bitwise-and-of-numbers-range/) [中等] • common-prefix-via-shift *（参考解法：仅 Python）*

### 进阶

- [LC 89 — Gray Code](https://leetcode.com/problems/gray-code/) [中等] • gray-code-formula *（参考解法：仅 Python）*
- [LC 1521 — Find a Value of a Mysterious Function Closest to Target](https://leetcode.com/problems/find-a-value-of-a-mysterious-function-closest-to-target/) [困难] • sliding-and-deduplication *（参考解法：仅 Python）*

LC 1521 是少见的、专为“逐位单调性”论证而设的困难题：以当前位置结尾的每个子数组 AND 只会清位，绝不会置位，所以往后带的不同的 AND 值最多只有 `log2(max(arr)) + 1` 个。这是速查手册式推理在大规模下的纯粹体现。

## 后续衔接

[XOR 模式](./01-xor-patterns.md) 从消去恒等式（`a ^ a = 0`）出发，在上面搭起单数那一族题，用 `x & -x` 当取位器，把两个唯一的数分到两组。[用位掩码枚举子集](../part-7-recursion-backtracking/02-subsets-combinations-permutations.md) 用本章“遍历置位位”一节里的循环来遍历一个掩码的置位位。[位掩码 DP](../part-9-dynamic-programming/14-bitmask-dp.md) 用同样的遍历形状，在 `O(2^n)` 个子集状态之间做转移。

第 11 部分之外，手册里这些恒等式也到处出现。树状数组用 `i += i & -i` 走索引。JDK 的 `HashMap` 用 `Integer.numberOfLeadingZeros` 配合消去最低置位位的恒等式，把容量向上取到 2 的幂。二分查找用无符号右移来算中点、避开溢出：Java 写 `lo + ((hi - lo) >>> 1)`，其他语言是同样的逻辑。

下一篇是 [XOR 模式](./01-xor-patterns.md)。

## 参考文献

[^1]: Stack Overflow, "cryptic comment in jdk source: // HD, Section 2-1", 2009, [stackoverflow.com/questions/1006626](https://stackoverflow.com/questions/1006626/cryptic-comment-in-jdk-source-hd-section-2-1).

[^2]: Anderson, S.E., "Bit Twiddling Hacks", Stanford Computer Graphics Laboratory, 1997-2005, [graphics.stanford.edu/~seander/bithacks.html](https://graphics.stanford.edu/~seander/bithacks.html).

[^3]: Knuth, D.E., *The Art of Computer Programming, Volume 4A: Combinatorial Algorithms, Part 1*, Addison-Wesley, 2011.

[^4]: Liu, N., "Algorithms behind Popcount," Nimrod's Coding Lab, 2022, [nimrod.blog/posts/algorithms-behind-popcount](https://nimrod.blog/posts/algorithms-behind-popcount/).

[^5]: Stack Overflow, "Why was 1 << 31 changed to be implementation-defined in C++14?", [stackoverflow.com/questions/26331035](https://stackoverflow.com/questions/26331035).

[^6]: The Go Authors, "The Go Programming Language Specification — Operators," [go.dev/ref/spec#Operators](https://go.dev/ref/spec#Operators).
