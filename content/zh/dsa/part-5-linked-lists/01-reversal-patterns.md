---
title: "反转模式"
description: "用 prev、curr、next 三个指针原地反转单链表。先快照、再覆盖的规则，以及迭代与递归两种写法——它们会作为基本操作出现在后面每一篇链表章节里。"
slug: 01-reversal-patterns
part: 5
chapter: 1
difficulty: intermediate
prerequisites:
  - part-5-linked-lists/00-linked-list-fundamentals
  - part-0-foundations/02-recursion-mental-model
date_updated: 2026-05-24
languages: [python, java, cpp, go]
canonical_test: LC-206
widgets: [w-13-linked-list-pointer-rewiring]
ladder:
  core: [LC-206, LC-92, LC-234]
  stretch: [LC-143]
  star: LC-206
---

# 反转模式

一条五个节点的单链表，`1 -> 2 -> 3 -> 4 -> 5`。面试官要的是同一批节点、顺序颠倒、一趟走完，除常数个指针之外不许多占存储。没见过这个技巧的读者会去借一个数组，把值抄出来，再抄回去。答案是对的，但要多付 `O(n)` 空间，而面试官的追问是“用 `O(1)` 做”。

这个追问才是本课的内容。原地反转单链表只有一种走法，它靠一条很小但毫不留情的规则：唯一告诉你下一步往哪走的指针，正是你必须覆盖的那个指针。先快照，再覆盖。顺序错一行，链表要么只剩一个节点，要么运行时在空指针解引用上崩掉。

## 为什么显而易见的做法不是正确答案

LC 206（Reverse Linked List）是这道题的招牌形态。输入：链表头。输出：反转后链表的头。约束：`0 <= n <= 5000`。题面直接对每位面试者提出追问：“A linked list can be reversed either iteratively or recursively. Could you implement both?” 底下那个没写出来的问题是：“迭代版请用 `O(1)` 空间。”

显而易见的做法是绕道数组：

```python
# 暴力：O(n) 时间、O(n) 额外空间 —— 这是我们准备替换掉的写法
def reverse_brute(head):
    values = []
    curr = head
    while curr is not None:
        values.append(curr.val)
        curr = curr.next
    curr = head
    for v in reversed(values):
        curr.val = v
        curr = curr.next
    return head
```

两趟遍历，一个数组。能跑。但它过不了本章的追问：数组是 `O(n)` 额外空间，在 5000 个节点的输入上，判题器的空间预算会在意这笔账。更糟的是，要求 `O(1)` 的面试官只会说一句“不用数组再做一遍”，而只会抄值的读者就无处可去了。

绕道数组之所以是错的，原因在结构上。单链表不让你往回走。每个节点只知道下一个节点，别的什么都不知道。所以想让节点 5 在反转后指向节点 4，*唯一*的办法是先沿 1 -> 2 -> 3 -> 4 -> 5 正向走一遍，然后在经过时完成重连。现在再看节点 1 上的重连本身。你想要 `n1.next = null`（n1 将成为新的尾节点）。可 `n1.next` 是你通往 n2 的唯一指针；一覆盖，后面的链表就消失了。

修法是多一个局部变量。在覆盖 `n1.next = null` *之前*，先把 `n1.next` 存进名为 `next` 的变量。这个快照才让你能继续往前走。一共三个指针，从不用四个。Sedgewick 与 Wayne 在《Algorithms》§1.3 讲的是同一个循环；CLRS 里正式要求它的那道习题是标准出处。[^1]

## 模式

三个局部指针——`prev`、`curr`、`next`——每轮四个子步骤。每一步只做一件事；顺序本身就是算法。

```python
def reverse_list_iterative(head):
    prev, curr = None, head
    while curr is not None:
        nxt = curr.next        # 1. 快照
        curr.next = prev       # 2. 重连（唯一的修改）
        prev = curr            # 3. 滑动 prev
        curr = nxt             # 4. 滑动 curr
    return prev
```

**解法** — Python（[Java](./01-reversal-patterns/lc-206/sol.java) · [C++](./01-reversal-patterns/lc-206/sol.cpp) · [Go](./01-reversal-patterns/lc-206/sol.go)）

```python
# LC 206. Reverse Linked List
from typing import Optional


class ListNode:
    def __init__(self, val: int = 0, nxt: "Optional[ListNode]" = None) -> None:
        self.val = val
        self.next = nxt


def reverse_list_iterative(head: Optional[ListNode]) -> Optional[ListNode]:
    """迭代版：prev/curr/next 三指针重连。

    进入循环时的不变式：严格位于 curr 左侧的每个节点都已完成重连，
    next 指针反向指向前一个节点；prev 是这段已反转前缀的新头；
    从 curr 开始的后缀结构未变。
    时间 O(n)，辅助空间 O(1)。
    """
    prev: Optional[ListNode] = None
    curr: Optional[ListNode] = head
    while curr is not None:
        nxt = curr.next
        curr.next = prev
        prev = curr
        curr = nxt
    return prev


def reverse_list_recursive(head: Optional[ListNode]) -> Optional[ListNode]:
    """递归版：先反转尾部，再重连 head.next.next = head。

    递归深度为 O(n)；CPython 默认上限是 1000，因此 n > ~900 时
    调用方必须提高 sys.setrecursionlimit()，或改用迭代版。
    时间 O(n)，调用栈上的辅助空间为 O(n)。
    """
    if head is None or head.next is None:
        return head
    new_head = reverse_list_recursive(head.next)
    head.next.next = head
    head.next = None
    return new_head
```

循环不变式说的，就是四个子步骤在运动中说的那件事：每轮迭代开始时，严格位于 `curr` 左侧的每个节点都已反向重连，`prev` 指着这段已反转前缀的头；从 `curr` 起的后缀结构完全未变。`curr` 变成 `null` 时循环结束，新的头就是退出时 `prev` 指向的节点。时间是 `O(n)`，因为每个节点恰好被访问一次、每次只做 `O(1)` 工作；辅助空间是 `O(1)`，因为链表之外的状态只有三个局部指针。

> [!IMPORTANT]
> **先快照，再覆盖。** 四个子步骤的顺序是 `snapshot, rewire, slide-prev, slide-curr`，不许调换，重连绝不能写在快照前面。重连会毁掉 `curr.next`；没有快照，下一轮读到的就是 `prev` 而不是原来的下一个节点，循环要么在一个节点之后终止，要么在空指针解引用上崩掉，而这个 bug 在本章的两节点测试（`[1, 2]`）打中之前一直是隐形的。

## 观察重连发生

这个交互组件在 `head = [1, 2, 3, 4, 5]` 上逐步演示 LC 206 的执行轨迹。共 23 步：循环体执行五次，每次拆成四个子步骤，再加三个边界状态（初始化、循环退出、返回）。先走一遍再往下读；快照与修改之间的依赖关系，动手感受比读文字更快。

> **Interactive widget:** [链表指针重连](../../../dsa/widgets/w-13-linked-list-pointer-rewiring.yml)（面板 `panel-b`）—— 在网站上渲染；YAML 规格描述的是各个关键帧。

*指针标记用的是 Wong 配色：prev 天蓝、curr 橙、next 黄。被反转的边在重连那一刻翻转箭头朝向——第 2、6、10、14、18 步就是那五次修改。*

组件里最显眼的子步骤是第 2 步：`n1.next` 从指向 `n2` 翻成指向 `null`。这一次边修改就是整个算法，其余全是记账。同样的动作在第 6、10、14、18 步依次出现，每次都让已反转前缀正好延长一个节点。到第 18 步链表已完全反转；第 19-22 步纯粹是收尾：跑完最后一轮迭代并返回 `prev`。

## 递归写法：面试官问起时

同一个操作，写成一次结构归纳。一路递归到尾部；往回走时，让每个节点的后继指向它自己。

```python
def reverse_list_recursive(head):
    if head is None or head.next is None:
        return head
    new_head = reverse_list_recursive(head.next)
    head.next.next = head      # 递归重连
    head.next = None           # 剪断正向边
    return new_head
```

`head.next.next = head` 这一行就是重连。递归调用返回后，`head.next` 仍指向原链表里 `head` 的后继（递归往下走时并没有改这条边）；那个后继的 `next` 现在应该指回 `head`，这正好就是 `head.next.next = head`。接着 `head.next = None` 剪断正向边，让新链表的尾节点正确终止。漏掉第二行，结果就是一条带环的链表：对 `[1, 2, 3]` 来说，`n2.next` 仍指向 `n3`，而 `n3.next` 现在指向 `n2`，于是 `n2 -> n3 -> n2 -> n3 -> ...` 在之后任何一次遍历里都会无限循环。

递归版是 `O(n)` 时间，但要 `O(n)` 的*栈*空间。最后这点才是坑。CPython 默认的 `sys.getrecursionlimit()` 是 1000，而 LC 206 的最坏输入是 `n = 5000`。不提上限直接提交递归版，判题器返回的是 `RecursionError` 运行时崩溃，不是错答案。[^2] 所以在意空间的面试里，迭代版才是标准答案；递归版的价值在于它是第二个心智模型，也是“两种都写给我看”的答案。

## 实例推演：LC 92（Reverse Linked List II）

LC 92（Reverse Linked List II，中等）把同一套重连扩展到一段连续子区间。输入：一条链表，外加 1 起始的位置 `left` 和 `right`。输出：`left .. right` 这段原地反转后的链表，区间外的不动。追问是“能一趟做完吗？”

对 `head = [1, 2, 3, 4, 5], left = 2, right = 4`，答案是 `[1, 4, 3, 2, 5]`。比 LC 206 多两个难点：头节点可能在反转段内，也可能不在（`left == 1` 的情形）；反转段结束后还得重新接上它左右两侧的部分。

[链表基础](./00-linked-list-fundamentals.md) 里的哑节点哨兵解决第一个难点。在下标 0 处多分配一个节点，让它的 `next` 指向真正的头；这样“反转段之前的那个节点”永远存在，即使 `left == 1` 也一样。第二个难点用一趟头插法解决：先用 `pre` 指针走到反转段的前一个节点，然后每轮把 `curr.next` 上的节点从原位置摘出来，插到已反转前缀的最前面。反转段的第一个节点原地不动，到最后自然成了这段的尾节点。

```python
def reverse_between(head, left, right):
    if head is None or left == right:
        return head

    dummy = ListNode(0, head)
    pre = dummy
    for _ in range(left - 1):
        pre = pre.next

    curr = pre.next
    for _ in range(right - left):
        moved = curr.next
        curr.next = moved.next
        moved.next = pre.next
        pre.next = moved

    return dummy.next
```

**解法** — Python（[Java](./01-reversal-patterns/lc-92/sol.java) · [C++](./01-reversal-patterns/lc-92/sol.cpp) · [Go](./01-reversal-patterns/lc-92/sol.go)）

```python
# LC 92. Reverse Linked List II
# 一趟头插法：哨兵先走 (left-1) 步，落在反转段之前；
# 然后做 (right-left) 轮，把每个新遇到的节点插到
# 已反转前缀的最前面。
from typing import Optional


class ListNode:
    def __init__(self, val: int = 0, nxt: "Optional[ListNode]" = None) -> None:
        self.val = val
        self.next = nxt


def reverse_between(
    head: Optional[ListNode], left: int, right: int
) -> Optional[ListNode]:
    """一趟头插法完成区间反转。时间 O(n)，空间 O(1)。"""
    if head is None or left == right:
        return head

    dummy = ListNode(0, head)
    pre = dummy
    for _ in range(left - 1):
        pre = pre.next  # type: ignore[assignment]

    # `curr` 是待反转段的第一个节点；它原地不动，
    # 最终成为该段的尾节点。每轮把 curr.next 摘出来，
    # 插到已反转前缀的最前面。
    curr = pre.next
    for _ in range(right - left):
        moved = curr.next
        curr.next = moved.next
        moved.next = pre.next
        pre.next = moved

    return dummy.next
```

在 `[1, 2, 3, 4, 5], left = 2, right = 4` 上走一遍。哨兵走完之后，`pre` 停在节点 1，`curr` 停在节点 2，链表仍然是 `dummy -> 1 -> 2 -> 3 -> 4 -> 5`。

第 1 轮。`moved = curr.next = 3`。把它摘出来：`curr.next = moved.next = 4`。再插到已反转前缀的最前面：`moved.next = pre.next = 2`，然后 `pre.next = moved = 3`。链表变成 `dummy -> 1 -> 3 -> 2 -> 4 -> 5`。节点 2 往后掉了一位；已反转前缀现在是 `3 -> 2`。

第 2 轮。`moved = curr.next = 4`（`curr` 还是节点 2，它从不移动）。摘出：`curr.next = 5`。插入：`moved.next = pre.next = 3`，然后 `pre.next = 4`。链表变成 `dummy -> 1 -> 4 -> 3 -> 2 -> 5`。

两轮，因为 `right - left = 4 - 2 = 2`。反转在反转段上一趟完成，`O(1)` 辅助空间的约定与 LC 206 相同。doocs/leetcode 上 Java、C++、Go 的参考解法都是这个头插法形状；上面那组四语言代码就照它写。[^3]

## 实际开销

| 变体 | 时间 | 空间 | 备注 |
|---|---|---|---|
| 迭代整表（LC 206） | `Theta(n)` | `O(1)` | 面试默认写法。三个局部指针；每个节点一次修改。 |
| 递归整表（LC 206） | `Theta(n)` | `O(n)` 栈 | CPython 深度约 1000 就到顶；LC 的 `n = 5000` 会撞墙。[^2] |
| 区间反转一趟（LC 92） | `O(n)` | `O(1)` | 哨兵走到位置 `left - 1`，再做 `right - left` 次头插。 |
| 反转作为子过程（LC 234、LC 143） | `O(n)` | `O(1)` | 快慢指针找中点，反转后半段，然后比较（LC 234）或归并（LC 143）。 |

迭代整表那版的 `O(1)` 辅助空间上界，正好是 CLRS 习题 10.2-7 *要求*的那个上界：那道习题要求一个 `Theta(n)` 时间的非递归过程来反转单链表，“除链表本身所需之外，不超过常数存储”。[^1] 三局部指针的解法就是满足它的标准做法。同一条约束延伸到 LC 92 和“反转作为子过程”的模式：这个家族里每道题都继承了 `O(1)` 这个天花板，因为每道题复用的都是同一个循环体。

## 模式失效的场景

**整表，迭代（LC 206）。** 标准形态。没有位置记账；重连从头到尾一视同仁。新的头就是循环退出时 `prev` 指向的节点。

**整表，递归（LC 206 追问）。** 同一个操作写成结构归纳。教科书喜欢这个心智模型；面试评分表不喜欢这个运行时画像。面试官要求两种都写时用它；在意空间就默认迭代。

**子区间，一趟（LC 92）。** 哨兵 + 头插法的变体。它自然推广到“每 K 个节点反转一次”；那个变体是 LC 25（Reverse Nodes in k-Group，困难），放在[每 K 个一组反转](./02-reverse-in-groups-k.md)，因为接缝记账（数 `k`、重接各组、处理尾部余数）本身就够一个教学单元。

**反转作为子过程（LC 234、LC 143）。** 反转只是一个阶段，不是整个算法。用快慢指针找中点，原地反转后半段，然后要么比较两半（LC 234，回文判定），要么把它们交错成一条输出（LC 143，重排）。讲环检测的那章接过快慢指针这套骨架；讲链表归并的那章接过交错这个阶段。

## 三个最容易踩的坑

> [!WARNING]
> **递归返回时忘了 `head.next = None`。** 做完 `head.next.next = head` 之后，原来的 `head.next` 这条边还没被覆盖——递归往下走时并没有改它。于是 `head.next` 仍指向下一个节点，而那个节点现在指回 `head`。之后任何遍历都会在这两个节点之间无限循环。修法是返回路径上成对的两行：先 `head.next.next = head`，再 `head.next = None`。AlgoMonster 的 LC 206 题解把它列为 Common Pitfall #1。[^4]

> [!WARNING]
> **LC 92 哨兵的差一错误。** 当 `left = 2` 时，实现者希望 `pre` 停在节点 1（反转段*之前*那个节点），但顺手写的 `for _ in range(left)` 更贴合 1 起始的心智模型，最后把 `pre` 落在节点 2 上——那是反转段的*第一个*节点。随后的插入重连了错的边，反转段整体差一位，返回错答案。修法是 `for _ in range(left - 1)`：哑节点算第 0 步，所以循环走 `left - 1` 次，不是 `left` 次。

> [!WARNING]
> **在修改之后才快照。** 把 `curr.next = prev` 写在 `next = curr.next` *前面*，是经典的顺序 bug。第一次重连之后，`curr.next` 已不再指向原来的下一个节点，它指向 `prev`（第一轮里是 `null`）。下一次推进读到的是错的指针，循环要么在一个节点之后终止，要么在空指针解引用上崩掉。四个子步骤必须按这个顺序执行：`next = curr.next; curr.next = prev; prev = curr; curr = next`。把循环体读成四个具名子步骤，并让组件把每一步单独渲染出来，依赖关系就一目了然。

## 题目梯度

### 核心（三题全做才算掌握本章）

- [LC 206 — Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) [简单] • linked-list-reversal-iterative-three-pointer ★
- [LC 92 — Reverse Linked List II](https://leetcode.com/problems/reverse-linked-list-ii/) [中等] • sub-range-reversal-by-position
- [LC 234 — Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list/) [简单] • linked-list-reversal-as-subroutine

### 进阶（可选）

- [LC 143 — Reorder List](https://leetcode.com/problems/reorder-list/) [中等] • linked-list-reversal-and-merge

LC 206 是本章的招牌题（★），也是四语言代码面板都通过其标准测试的那一道。核心三题只覆盖简单/中等；自然的困难变体 LC 25（Reverse Nodes in k-Group）归[每 K 个一组反转](./02-reverse-in-groups-k.md)所有，因为正是接缝记账让它成为困难题。LC 234 在 LC 上官方标简单，但最优的 `O(1)` 空间写法合并了三个阶段（快慢指针找中点、反转后半段、同步走并比较），第一次读的人会觉得更接近中等。

## 后续衔接

[每 K 个一组反转](./02-reverse-in-groups-k.md) 把 LC 206 的循环体套进一层外部计数里。重连一样，记账更难：先数出 `k` 个节点，把它们当作一个单位反转，再把反转后的组接回原链表，重复。那一章的组件面板与本章共用渲染器和指令集；按 5.0 -> 5.1 -> 5.2 的顺序读下去，同一个组件会依次推进三个难度级别。

“反转作为子过程”这一族亲属则通向别处。LC 234 和 LC 143 依赖的快慢指针中点扫描是它自己的模式；那套骨架属于[环检测与快慢指针](./03-cycle-detection.md)。LC 143 收尾的交错归并阶段，则推广到[链表归并](./04-merging-linked-lists.md)里两条有序链表的归并。

## 参考文献

[^1]: Cormen, Leiserson, Rivest, Stein, *Introduction to Algorithms*, 4th ed., MIT Press, 2022. §10.2, Exercise 10.2-7.

[^2]: Python 3 docs, `sys.setrecursionlimit`. <https://docs.python.org/3/library/sys.html#sys.setrecursionlimit>

[^3]: doocs/leetcode, LC 92 reference solutions. <https://github.com/doocs/leetcode/blob/main/solution/0000-0099/0092.Reverse%20Linked%20List%20II/README_EN.md>

[^4]: AlgoMonster, LC 206 editorial. <https://algo.monster/liteproblems/206>
