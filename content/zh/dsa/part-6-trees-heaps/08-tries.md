---
title: "Trie"
description: "子节点映射加终止标记的节点，让前缀遍历的时间只取决于查询长度，为 O(L)。自动补全与 IP 路由这一族问题里，前缀结构就是全部要害。"
slug: 08-tries
part: 6
chapter: 8
difficulty: intermediate
prerequisites:
  - part-6-trees-heaps/00-binary-tree-fundamentals
  - part-1-linear-data-structures/03-hash-maps
date_updated: 2026-05-25
languages: [python, java, cpp, go]
canonical_test: LC-208
widgets: [w-20-trie]
ladder:
  core: [LC-208, LC-720, LC-212]
  stretch: [LC-211, LC-1268]
  star: LC-208
---

# Trie

iPhone 键盘上每敲一个键，都在走一棵 Trie。浏览器自动补全的每个 URL、Linux 内核转发的每条 IP 路由、你此刻正在打字的编辑器里拼写检查器校验的每个单词，同样如此。这个数据结构可追溯到 René de la Briandais 1959 年的论文，以及 Edward Fredkin 1960 年的论文（Fredkin 从 "re*trie*val" 里造出了这个名字），而它今天回答的仍是当年那个问题：给定一组字符串，用与查询长度成正比、而不是与集合大小成正比的时间回答前缀查询。

哈希集合能在 O(L) 时间内回答“`cat` 是不是一个单词”，L 是 `cat` 的长度。它做不到的是——除非把每个键都重扫一遍——回答*“列出所有以 `ca` 开头的单词”*。这正是 Trie 填补的缺口。[^1] 填补它的代价是结构性的：一棵树的边是字符，路径拼出前缀，终止标记则表示“你走到的这条路径是一个完整单词”。插入和查找都是每走一个字符就下一个节点；词典里的冗余坍缩成共享的链。*Algorithms* 4e §5.2 把这种结构叫 R 路 Trie；LeetCode 题面叫它前缀树（prefix tree）；两者指的是同样六行代码和同样三个操作。[^2]

两道真正兑现价值的题是 LC 211（通配符查找）和 LC 212（二维网格上的单词搜索）。LC 212 更难。不用 Trie 的逐词 DFS 在 LC 最狠的用例上直接超时；Trie 加速的 DFS 靠一次空子节点检查就剪掉整片模式子树，同一道题立刻变得轻松。

## 结构里有什么

Trie 是一棵字符树。每个节点持有一个按字符索引的子节点数组（或哈希映射），外加一个布尔量 `is_end`，表示“有一个被插入的单词恰好在这里结束”。根节点不带字符；它的子节点代表字母表的首字母；往下走一条边就消耗输入的一个字符。从根到任意节点的路径*就是*它所代表的前缀。

插入操作沿着新单词的字符所指的路径走，路径不存在的地方就分配节点，最后在终点节点上把 `is_end` 打开。查找走同样的路径，走完问两个问题：是否到达了一个非空节点，以及那里的 `is_end` 是否置位？两个都为真才返回 true。`startsWith` 走同样的路径，但只问第一个问题。

```python
class TrieNode:
    __slots__ = ("children", "is_end")
    def __init__(self):
        self.children = [None] * 26
        self.is_end = False

class Trie:
    def __init__(self):
        self.root = TrieNode()

    def insert(self, word):
        node = self.root
        for ch in word:
            i = ord(ch) - ord("a")
            if node.children[i] is None:
                node.children[i] = TrieNode()
            node = node.children[i]
        node.is_end = True

    def search(self, word):
        node = self._walk(word)
        return node is not None and node.is_end

    def startsWith(self, prefix):
        return self._walk(prefix) is not None

    def _walk(self, s):
        node = self.root
        for ch in s:
            i = ord(ch) - ord("a")
            nxt = node.children[i]
            if nxt is None:
                return None
            node = nxt
        return node
```

**题解** — Python ([Java](./08-tries/lc-208/sol.java) · [C++](./08-tries/lc-208/sol.cpp) · [Go](./08-tries/lc-208/sol.go))

```python
# LC 208. Implement Trie (Prefix Tree)
from typing import List, Optional


class TrieNode:
    __slots__ = ("children", "is_end")

    def __init__(self) -> None:
        self.children: List[Optional["TrieNode"]] = [None] * 26
        self.is_end: bool = False


class Trie:
    def __init__(self) -> None:
        self.root = TrieNode()

    def insert(self, word: str) -> None:
        node = self.root
        for ch in word:
            idx = ord(ch) - ord("a")
            if node.children[idx] is None:
                node.children[idx] = TrieNode()
            node = node.children[idx]
        node.is_end = True

    def search(self, word: str) -> bool:
        node = self._walk(word)
        return node is not None and node.is_end

    def startsWith(self, prefix: str) -> bool:
        return self._walk(prefix) is not None

    def _walk(self, s: str) -> Optional[TrieNode]:
        node = self.root
        for ch in s:
            idx = ord(ch) - ord("a")
            nxt = node.children[idx]
            if nxt is None:
                return None
            node = nxt
        return node
```

从代码里能读出三件事。遍历逻辑在 `search` 和 `startsWith` 之间共享；`_walk` 替两者干活。两者唯一的功能差异是末尾那个 `node.is_end` 检查，其余完全相同。每个操作的迭代次数恰好是 `len(input)`，每处理一个字符只需一次数组索引和一次空值检查。没有递归，没有平衡，没有旋转。是数据的形状在干活。

这个交互组件演示了往空 Trie 里依次插入 `car`、`cat`、`card`，然后遍历前缀 `ca`。共享的 `ca` 链就是本章的中心图像：

> **Interactive widget:** [Trie — 插入 + 前缀遍历 + 词频自动补全](../../../dsa/widgets/w-20-trie.yml) (panel `default`) — 在网站上渲染；YAML 规格描述了关键帧。

*静态说明：Trie 起初只有一个孤零零的根节点。插入 car 建出链 root → c → a → r，并把 r 标记为 is_end。插入 cat 复用路径上已有的 c 和 a 节点，然后在 n_ca 上分出一条 t 分支；只分配了一个新节点。插入 card 复用 c、a、r；r 早已因 car 置了 is_end，现在又多了一个 is_end 的 d 子节点（对应 card）。对 ca 的遍历走两条边，报告 startsWith = true；开销与 Trie 里存了多少单词无关。*

让每个操作都是 O(L) 的那个观察只有一条：路径长度等于输入长度。分支因子 `R` 只作为常数进入每条边的索引步骤（数组：O(1)；哈希映射：平均 O(1)），并不影响路径长度。[^1] 哈希表的查找也是 Ω(L)，因为哈希函数要读完键的每个字符，所以 Trie 的 O(L) 在单点查找上*并不*比哈希渐近更慢；它只是*额外*能回答哈希答不了的前缀查询。[^3]

## `is_end` 是什么，不是什么

新手写 Trie 时最容易引入的 bug，是把两个并不相同的概念揉成一个。

`is_end` 是一个标记，表示“有一个被插入的单词在这里结束”。节点是否有子节点，是关于这个节点在结构中位置的另一件事实。插入 `car` 和 `card` 之后，节点 `n_car` 既满足 `is_end = true`，又为 `n_card` 持有一个 `d` 子节点。把 `is_end` 当成“叶子”的同义词，等 `card` 进了词典，`search("car")` 就会悄悄出错；或者反过来，实现里该判断非空节点却一直在测 `is_end`，`startsWith("car")` 就坏了。

> [!IMPORTANT]
> `search` 检查 `is_end`。`startsWith` 不检查。把 `search` 复制到 `startsWith` 里，忘了删掉 `is_end` 判断，那么 `insert("apple")` 之后 `startsWith("a")` 会返回 false——LC 208 的判题器能抓到这个错误答案，不判题的读者却抓不到。LC 208 官方题解专门点了这一点，说这正是为什么共享的辅助函数是 `searchPrefix`，而 `search` 在它之上再加一层 `is_end` 检查。[^4]

还有一条不变量值得说一次。插入 `apple` 得到链 `a → p → p → l → e`，其中*第一个* `p` 和*第二个* `p` 是两个不同的节点。单词里的重复字符不会合并；Trie 的分支是按位置分的，不是按字母分的。把代码读成：输入的每个字符都在*当前节点的 R 个子节点里选择下一个桶*，跟上一个字符选了谁毫无关系。把这一点内化，Trie 就变成常规操作了。

## 选择子节点容器

上面展示的 R 路数组是教科书默认做法，也是固定小写英文字母表下的正确选择。换成别的约束，就该换别的形状。

每节点一个 26 指针数组很快：每次下行一次带索引的解引用，连续的 26 槽行带来完整的缓存局部性，代价是不管节点只有 1 个子节点还是 26 个，都要按 `R` 个指针占内存。一棵 10 万节点、指针 8 字节的 Trie，光子节点数组就是 20 MB，其中大部分是空槽。Sedgewick 与 Wayne 在 R 路 Trie 的分析里把这个权衡讲得很清楚：纯数组节点下行最快，代价是每节点 `R` 个指针；基于映射的节点把内存降到每节点 `O(d_v)`，`d_v` 是实际用到的不同子字符数，代价是每条边做一次哈希而不是一次带索引的读取。[^3]

| 容器 | 每条边开销 | 每节点内存 | 适用场景 |
|---|---|---|---|
| `children[R]`（R 路数组） | O(1) 索引读取 | R 个指针，稠密 | 小的固定字母表（小写字母、DNA、IPv4 nibble）；LC 208 默认 |
| 哈希映射子节点 | O(1) 哈希，平均 | 实际的 `O(d_v)` 个子节点 | Unicode、任意字节、大或未知字母表 |
| 三叉搜索 Trie | O(log R) 下行 | 每节点 3 个指针 | 需要有序遍历又不想浪费 R 数组；Sedgewick 的 `algs4` 附带 TrieST.java[^3] |

生产实现会拐得更远。Linux 内核的 IP 路由表用的是*压缩* Trie（radix tree），把任意一条单子节点链合并成一条带多字符串标签的边；内存从每个字符 `O(R)` 降到每个分支点 `O(R)`，查找代价仍以比较的总字符数为界。[^2] Donald Morrison 1968 年的 PATRICIA 论文提出压缩 Trie 正是为了解决这个：又长又稀疏分支的键在未压缩形式下浪费巨量内存。本章的 CORE 题（LC 208、LC 720、LC 212）用 R 路数组既正确又好推理；压缩变体在键很长、前缀共享很稀疏时才显出价值，下面的变体表给出了交叉索引。

## 当遍历不再是一条路径

LC 208 讲的是机制。真正兑现价值的两道题是 LC 211 和 LC 212。两者都保持 Trie 的结构不变，只是把线性遍历换成递归搜索。

### LC 211：通配符与树形遍历

给定一串 `addWord` 和 `search` 调用，其中查询可能含有匹配任意单个字母的 `.`，返回词典里是否存在*任何*匹配合该查询的单词。

信号很明确：单字符通配符把遍历从“只沿一条边走”变成“把每个非空子节点都试一遍”。遍历不再是一条路径，而是以当前节点为根的 Trie 子树上的深度优先搜索。

```python
def search(self, word):
    return self._dfs(self.root, word, 0)

def _dfs(self, node, word, i):
    if i == len(word):
        return node.is_end
    ch = word[i]
    if ch == ".":
        for child in node.children:
            if child is not None and self._dfs(child, word, i + 1):
                return True
        return False
    nxt = node.children[ord(ch) - ord("a")]
    if nxt is None:
        return False
    return self._dfs(nxt, word, i + 1)
```

**题解** — Python ([Java](./08-tries/lc-211/sol.java) · [C++](./08-tries/lc-211/sol.cpp) · [Go](./08-tries/lc-211/sol.go))

```python
# LC 211. Design Add and Search Words Data Structure
# [addWord("bad"), addWord("dad"), addWord("mad"), search("pad")=False,
#  search("bad")=True, search(".ad")=True, search("b..")=True] 通过。
# 通配符 '.' 匹配任意一个小写字母；遇到通配符的那一步递归
# 遍历所有非空子节点。LC 211 把查询里的点限制在 2 个，
# 因而最坏情况是每次搜索 26^2 = 676 条路径
#。
from typing import List, Optional


class _Node:
    __slots__ = ("children", "is_end")

    def __init__(self) -> None:
        self.children: List[Optional["_Node"]] = [None] * 26
        self.is_end: bool = False


class WordDictionary:
    def __init__(self) -> None:
        self.root = _Node()

    def addWord(self, word: str) -> None:
        node = self.root
        for ch in word:
            idx = ord(ch) - ord("a")
            if node.children[idx] is None:
                node.children[idx] = _Node()
            node = node.children[idx]
        node.is_end = True

    def search(self, word: str) -> bool:
        return self._dfs(self.root, word, 0)

    def _dfs(self, node: _Node, word: str, i: int) -> bool:
        if i == len(word):
            return node.is_end
        ch = word[i]
        if ch == ".":
            for child in node.children:
                if child is not None and self._dfs(child, word, i + 1):
                    return True
            return False
        idx = ord(ch) - ord("a")
        nxt = node.children[idx]
        if nxt is None:
            return False
        return self._dfs(nxt, word, i + 1)
```

两个分支就是整个算法。具体字符：下到那一个子节点，或者失败。通配符：把每个存在的子节点都试一遍；任一子树里找到匹配就成功。

LC 211 的最坏情况是 `O(R^k)`，`k` 是点的个数：一个全由点组成的查询打在一棵 `R` 分支的 Trie 上，会把整棵子树展开。题面把查询限制在至多 2 个点，最坏情况因此被压到每次搜索 `26^2 = 676` 条路径。[^5] 这个上限是结构性的，不是可选项。去掉它，算法就需要带后缀链接的 Aho-Corasick 式自动机，那是另一章，而且难得多。[^6]

### LC 212：把 Trie 当作剪枝预言机

给定一个字母二维网格和一个单词列表，返回所有能在网格中沿相邻格子走出来（同一个单词内不重复使用格子）的单词。朴素解法是每个单词跑一次 DFS：对词典里每个词，试遍每个起点，在网格里走。词典三万词、单词最长 10 个字符、棋盘 12×12，这种打法到处超时。LC 212 就是专门设计来让它失败的。

Trie 加速版把循环倒过来。先把所有单词插进 Trie。然后在棋盘上跑*一次* DFS，同步推进格子指针和 Trie 节点指针。每一步，棋盘上的下一个字符必须等于要下行的边；Trie 子节点缺失就是立刻回溯的信号，一次性剪掉共享那个不可能前缀的*所有*单词。

```python
def findWords(self, board, words):
    root = TrieNode()
    for w in words:
        node = root
        for ch in w:
            node = node.children.setdefault(ch, TrieNode())
        node.word = w  # 在终止节点上存下这个单词

    rows, cols = len(board), len(board[0])
    found = []

    def dfs(r, c, parent):
        ch = board[r][c]
        node = parent.children.get(ch)
        if node is None:
            return                       # 剪枝
        if node.word:
            found.append(node.word)
            node.word = ""               # 抑制重复发现
        board[r][c] = "#"
        for dr, dc in ((-1, 0), (1, 0), (0, -1), (0, 1)):
            nr, nc = r + dr, c + dc
            if 0 <= nr < rows and 0 <= nc < cols and board[nr][nc] != "#":
                dfs(nr, nc, node)
        board[r][c] = ch
        if not node.children:
            parent.children.pop(ch, None) # 顺手缩小 Trie

    for r in range(rows):
        for c in range(cols):
            dfs(r, c, root)
    return found
```

**题解** — Python ([Java](./08-tries/lc-212/sol.java) · [C++](./08-tries/lc-212/sol.cpp) · [Go](./08-tries/lc-212/sol.go))

```python
# LC 212. Word Search II
# board=[["o","a","a","n"],["e","t","a","e"],["i","h","k","r"],
#        ["i","f","l","v"]], words=["oath","pea","eat","rain"]
# 返回 ["oath", "eat"]。
#
# Trie 加速的回溯：把所有单词插入一棵以棋盘字符为键的 Trie；
# 从每个格子出发做 DFS，同时推进格子指针和 Trie 节点指针。
# Trie 子节点缺失就是立刻回溯的信号——共享该前缀的整棵模式
# 子树在 O(1) 内被剪掉。找到单词后把 is_end 置为 False 以抑制
# 重复发现；再自底向上剪掉叶子节点，让 Trie 随着单词耗尽
# 而收缩。
from typing import Dict, List


class _Node:
    __slots__ = ("children", "word")

    def __init__(self) -> None:
        self.children: Dict[str, "_Node"] = {}
        self.word: str = ""


class Solution:
    def findWords(self, board: List[List[str]], words: List[str]) -> List[str]:
        root = _Node()
        for w in words:
            node = root
            for ch in w:
                node = node.children.setdefault(ch, _Node())
            node.word = w

        rows, cols = len(board), len(board[0])
        found: List[str] = []

        def dfs(r: int, c: int, parent: _Node) -> None:
            ch = board[r][c]
            node = parent.children.get(ch)
            if node is None:
                return
            if node.word:
                found.append(node.word)
                node.word = ""  # 抑制重复发现
            board[r][c] = "#"  # 标记已访问
            for dr, dc in ((-1, 0), (1, 0), (0, -1), (0, 1)):
                nr, nc = r + dr, c + dc
                if 0 <= nr < rows and 0 <= nc < cols and board[nr][nc] != "#":
                    dfs(nr, nc, node)
            board[r][c] = ch  # 恢复
            # 剪掉死枝。
            if not node.children:
                parent.children.pop(ch, None)

        for r in range(rows):
            for c in range(cols):
                dfs(r, c, root)
        return found
```

三处细节值这几行代码。把单词字符串存在终止节点上（而不只是 `is_end`），DFS 就能直接报告命中，不用重建走过的路径。命中后置 `node.word = ""`，抑制那些本会堆积起来的重复命中——DFS 从不同路径反复抵达同一个终止节点。自底向上剪掉叶子节点（当前节点没有剩余子节点时 `parent.children.pop(ch, None)`）让 Trie 随单词被耗尽而缩小，后续的 DFS 扫描面对的就是更小的结构。[^7]

LC 211 和 LC 212 之间的不对称值得点出来。LC 211 里 Trie 是承载答案的结构，搜索的返回值取决于里面有什么。LC 212 里 Trie 是一个*前瞻剪枝器*，被服务的那次搜索，状态活在网格里。两个角色不同，而“Trie 可以当剪枝器”是整个竞赛编程里题目模式匹配中杠杆率最高的模式之一。Aho-Corasick 通过给 Trie 加后缀链接，把 LC 212 的思路推广到多模式文本搜索。[^6]

## 实际开销

| 操作 | 时间 | 空间（每节点） | 备注 |
|---|---|---|---|
| `insert(word)` | O(L) | O(R) 数组；O(d_v) 哈希映射 | 每个字符：一次索引 + 一次空值检查 + 至多一次分配。[^1][^3] |
| `search(word)` | O(L) | 不变 | 同样的遍历，末尾多一次 `is_end` 检查。[^1] |
| `startsWith(prefix)` | O(L) | 不变 | 只遍历，不检查 `is_end`。[^1] |
| 建树（n 个单词，总字符数 m） | O(m) | 最坏 O(mR) 数组；O(m) 哈希映射 | 所有插入合计。 |
| LC 211 含 k 个点的搜索 | 最坏 O(R^k) | 不变 | 被 LC 的“至多 2 个点”限制压到 `26^2 = 676`。[^5] |
| LC 212 的 Trie 加速 DFS | 最坏 O(M·N · 4 · 3^(L−1)) | O(K) 的 Trie 规模 | M·N 个格子，L = 最长单词长度，K = 所有单词的字符总数；实践中剪枝把常数因子砍掉一大截。[^7] |

记号：`L` = 该操作输入串的长度；`R` = 字母表大小（小写字母 26，ASCII 256，Unicode 更大）；`m` = Trie 里所有键长之和；`d_v` = 节点 v 上不同的子字符数。

遍历长度的界是结构性的：Trie 深度等于最长键，每个操作按输入字符数逐个触节点。分支因子只进入每条边的开销（数组 O(1)，哈希平均 O(1)），不进入路径长度。[^1] 在 R 路数组形式下 Trie *很吃内存*，这一点没必要假装不是；如果你的词典键很长且分支稀疏，生产答案是 radix tree。[^2]

### 值得了解的模式变体

| 变体 | 何时该用 | 备注 |
|---|---|---|
| **R 路数组**（本章默认） | 小的固定字母表（小写字母、DNA、十六进制） | 每条边一次带索引读取；完整的缓存局部性 |
| **哈希映射子节点** | Unicode、任意字节、未知字母表 | 每条边一次哈希；内存从 R 降到实际的 `d_v` |
| **压缩 Trie / radix tree**（Patricia） | 键长、分支稀疏、内存受限的系统（IP 路由表、文件路径索引） | 单子节点链合并成多字符边标签；Morrison 1968。[^2] |
| **三叉搜索 Trie** | 想要有序遍历，下行 O(log R)，内存适中 | Sedgewick 的 `algs4` 附带 TrieST.java；节点有 lt/eq/gt 三个子节点。[^3] |
| **DAWG**（有向无环词图） | 静态词典；除前缀外还想共享*后缀* | 同一节点可被多条路径到达；比 Trie 更省内存，代价是不可变更 |
| **后缀树** | 子串查询，而不是前缀查询（另一个问题） | 一个字符串所有后缀构成的 Trie；用 Ukkonen 算法 O(n) 建出 |
| **Aho-Corasick** | 文本中的多模式子串搜索 | Trie + 失败链接；LC 212 思路的推广。[^6] |

压缩 Trie 重要到值得单独给出交叉索引，但它与 R 路数组的常数比较取决于实现，超出本章范围；把它当成内存吃紧时的生产兜底。DAWG 和后缀树是完全不同的问题，只是碰巧共用 Trie 这具“字符之树”的骨架。

## 常见误区

> [!WARNING]
> **把 `is_end` 和“是叶子”搞混。** 写 `if node.is_end: it has no children`，或者认定只有叶子才能是 `is_end`。插入 `car` 和 `card` 之后，`n_car` 既 `is_end = true`，又有一个 `d` 子节点。把 `is_end` 和“有子节点”当成彼此正交的两个标记。`search` 检查前者；`startsWith` 两者都不检查。[^4]

> [!WARNING]
> **忘了 `startsWith` 不看 `is_end`。** 把 `search` 复制进 `startsWith` 却留着 `is_end` 检查，那么当 `insert("apple")` 是唯一一次插入时，`startsWith("a")` 返回 false。遍历明明到达了非空节点，而这恰恰正是 `startsWith` 该返回 true 的时刻。[^4]

> [!WARNING]
> **按错的字母表给子节点数组定尺寸。** 输入里含大写字母或数字却用 `[26]` 数组，`idx = ord('A') - ord('a') = -32`，索引到垃圾位置。纯小写输入却用 `[128]` 数组，浪费约 5 倍于所需的内存。读清约束；LC 208/211/212/720 用 `[26]`，字母表是 Unicode 或未知时换哈希映射。[^3]

> [!WARNING]
> **在 LC 212 上逐词 DFS。** 遍历单词列表、每个词各跑一次网格 DFS 是 `O(words × cells × 4^L)`，在 LC 的困难用例上超时。Trie 加速版是在棋盘上跑一次 DFS，词典坍缩成剪枝预言机。在 LC 的时间预算下，这两种做法不等价。[^7]

> [!WARNING]
> **LC 212 命中后跳过 `is_end = false` 这一步。** DFS 一旦抵达终止节点并记录了单词，若仍让 `is_end = true`（或者在上面的变体里让 `node.word` 保持非空），同一个词就会从每个能到达它的新起点被反复找到。答案集合会去重；工作量不会。第一次找到该词时就清掉标记。[^7]

## 题目梯度

### 核心（三题全做才算掌握本章）

- [LC 208 — Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) [中等] • trie-fundamentals-insert-search-startswith ★
- [LC 720 — Longest Word in Dictionary](https://leetcode.com/problems/longest-word-in-dictionary/) [简单] • trie-bfs-or-sort-prefix-incremental
- [LC 212 — Word Search II](https://leetcode.com/problems/word-search-ii/) [困难] • trie-accelerated-backtracking-on-grid

### 进阶（可选）

- [LC 211 — Design Add and Search Words Data Structure](https://leetcode.com/problems/design-add-and-search-words-data-structure/) [中等] • trie-with-wildcard-dfs
- [LC 1268 — Search Suggestions System](https://leetcode.com/problems/search-suggestions-system/) [中等] • trie-prefix-with-sorted-collection

LC 211 被放在进阶里，尽管正文给了它自己的实例推演，原因是通配符机制是 LC 208 的*自然延续*，而不是一项独立技能。大多数把 LC 208 做对的读者，下一次坐下就能做对 LC 211。

## 后续衔接

[用回溯做 Word Search II](../../../dsa/part-7-recursion-backtracking/05-word-search.md) 讲 LC 79（单词版），并且独立处理递归形状；让词典版（LC 212）变得可解的 Trie 剪枝则留在本章。[KMP 与字符串模式匹配](../../../dsa/part-12-strings-pattern-matching/02-kmp.md) 是关心多模式文本搜索的读者的自然下一步。Aho-Corasick 就是一棵外挂了失败链接的 Trie，而你现在能写出的 R 路 Trie 正是它的基底。

## 参考文献

[^1]: Wikipedia contributors, "Trie," last edited March 2026, [en.wikipedia.org/wiki/Trie](https://en.wikipedia.org/wiki/Trie).

[^2]: Wikipedia 的 "Radix tree" 条目记录了压缩 Trie 变体，并把功劳归于 Donald R. Morrison, "PATRICIA — Practical Algorithm to Retrieve Information Coded in Alphanumeric," *Journal of the ACM* 15(4), October 1968, pp. 514-534, doi:[10.1145/321479.321481](https://doi.org/10.1145/321479.321481).

[^3]: Robert Sedgewick and Kevin Wayne, *Algorithms*, 4th ed., Addison-Wesley, 2011, §5.2 Tries. 在线补充材料见 [algs4.cs.princeton.edu/52trie](https://algs4.cs.princeton.edu/52trie/).

[^4]: LeetCode 官方题面与题解，LC 208, "Implement Trie (Prefix Tree)," 难度 Medium。经 NeetCode 的 [implement-prefix-tree 题目页](https://neetcode.io/problems/implement-prefix-tree/question) 交叉确认。

[^5]: LeetCode 211, "Design Add and Search Words Data Structure," 约束页见 [leetcode.com/problems/design-add-and-search-words-data-structure](https://leetcode.com/problems/design-add-and-search-words-data-structure/)

[^6]: cp-algorithms, "Aho-Corasick algorithm," last updated April 2025, [cp-algorithms.com/string/aho_corasick.html](https://cp-algorithms.com/string/aho_corasick.html).

[^7]: AlgoMonster 对 LC 212, "Word Search II" 的题解，见 [algo.monster/liteproblems/212](https://algo.monster/liteproblems/212)
