# 1. 概念导入

想象你在一个岔路口，路牌写着"读 'a' 后可以去房间 A 或房间 B"。这就是 NFA 的"不确定性"：同一输入下，你可能走到多个不同的状态。NFA 解决的是"是否存在一条合法路径"的问题——只要 **存在一条** 能走到接受态的路径，输入就被接受。NFA 比 DFA 更"紧凑"（更少的状态就能描述同样的语言），但模拟起来更慢。

# 2. 核心原理

## 2.1 NFA 的定义

NFA（Nondeterministic Finite Automaton）也是五元组 (Q, Σ, δ, q₀, F)，与 DFA 的唯一区别在转移函数 δ：
- **DFA 的 δ**：Q × Σ → **Q**（单值）
- **NFA 的 δ**：Q × Σ → **2^Q**（值集——可能到达多个状态）

从状态 q 读输入 a，NFA 告诉你"可能去 {q₁, q₂, q₃} 中的任意一个"。只要有一条路径最终到达接受态，整个输入就被接受。

```python
# NFA 的转移表：同一状态 + 同一输入可对应多个下一状态
nfa = {
    (0, 'a'): {1, 2},   # 不确定性：读 'a' 可能去 1 或 2
    (0, 'b'): {0},
    (1, 'b'): {3},
    (2, 'b'): {3},
}
# DFA 对比：同一输入只有一个下一状态
dfa = {
    (0, 'a'): 1,        # 确定性：读 'a' 只能去 1
    (0, 'b'): 0,
    (1, 'b'): 2,
}
```

## 2.2 DFA 与 NFA 的对比

```
DFA:      'a'               NFA:      'a'
        q0 ──────► q1              q0 ──────► q1
                                    q0 ──────► q2
```

| 对比维度 | NFA | DFA |
|---------|-----|-----|
| 确定性 | 不确定，同一输入可走多条路 | 完全确定，一条路走到底 |
| 状态数 | 少（对 n 字符模式约 O(n)） | 可能多（最坏 O(2ⁿ)） |
| 从正则构造 | 直接、自然（Thompson 构造法） | 需先构造 NFA 再子集化 |
| 模拟速度 | O(n × k)，k 为状态数 | O(n)，确定性无分支 |
| 表达能力 | 与 DFA **等价**（正则语言） | 与 NFA **等价** |

> 本质：NFA 和 DFA 表达能力等价，但 NFA 更"紧凑"，DFA 更"快"。

## 2.3 ε 转移（空转移）

NFA 还允许一种特殊转移：**ε（空）转移**——不消耗任何字符就能跳转到另一个状态。这使得构造 NFA 更加灵活。

```
       ε                    从 q0 可以不读字符跳到 q1，
  q0 ──────► q1             也可以读 'a' 到 q2。
  │
  │ 'a'
  ▼
  q2
```

```python
# 带 ε 转移的 NFA：用空字符串 '' 代表 ε
nfa_with_eps = {
    (0, ''):  {1},       # 不消耗字符，从 0 跳到 1
    (0, 'a'): {2},
    (1, 'b'): {3},
    (2, 'c'): {3},
}
```

## 2.4 ε 闭包

模拟 NFA 的第一步：计算 ε 闭包——从某些状态出发，只通过 ε 转移能到达的 **所有** 状态集合。

```python
def epsilon_closure(nfa, states):
    """计算 states 中所有状态通过 ε 转移能到达的状态集合

    为什么用栈（DFS）：ε 转移可以链式发生，需要持续探索新状态
    """
    stack = list(states)
    closure = set(states)
    while stack:
        s = stack.pop()
        for ns in nfa.get((s, ''), set()):
            if ns not in closure:
                closure.add(ns)
                stack.append(ns)
    return closure

# 示例：从 {0} 出发的 ε 闭包
print(epsilon_closure(nfa_with_eps, {0}))  # {0, 1}
```

## 2.5 NFA 的模拟

NFA 的模拟不追踪单一路径，而是追踪 **一组可能的状态**——所有分支并行探索：

```python
def nfa_accept(nfa, text, start=0):
    """为什么维护状态集：所有可能路径并行探索，避免回溯指数爆炸"""
    states = epsilon_closure(nfa, {start})
    for ch in text:
        next_states = set()
        for s in states:
            next_states |= nfa.get((s, ch), set())
        states = epsilon_closure(nfa, next_states)
        if not states:
            return False
    return 3 in states
```

这种"状态集模拟"本质上就是在模拟一个 DFA——状态集是有限的（最多 2^{|Q|} 种），这正是 **子集构造法** 的原理。

# 3. 复杂度与适用场景

| 维度 | 指标 |
|------|------|
| 时间复杂度 | O(n × k)，n 为输入长度，k 为 NFA 状态数（最坏 O(n·2^{|Q|})） |
| 空间复杂度 | O(|Q|²)，转移表存储 |
| 回溯法最坏 | O(2ⁿ)，指数爆炸——这正是状态集模拟要避免的 |

**适用场景：**
- 正则表达式引擎（标准流程：RE → NFA → DFA）
- 模式数量多但希望自动机紧凑时（如文本编辑器的高亮规则）
- 语言的描述/构造阶段（工程师写正则，机器负责转 DFA）

**不适用场景：**
- 性能敏感的生产环境（通常转成 DFA 再执行）
- 输入流极长的实时处理（状态集可能膨胀）

# 4. 常见变体

## 4.1 ε-NFA
带 ε 转移的 NFA 标准形式。Thompson 构造法从正则构造出 ε-NFA，再通过 ε 闭包消除 ε 边得到普通 NFA。

## 4.2 多起始态 NFA
允许多个起始状态，相当于从多个起点"并行"开始匹配，在同时查找多个关键词时很实用。

# 5. 经典题目：简单正则表达式匹配

## 问题描述

实现支持 `.` 和 `*` 的正则匹配函数：
- `.` 匹配任意单个字符
- `*` 匹配零个或多个前面的字符
- 匹配必须覆盖 **整个** 字符串（而非子串）

## 思路分析

这是一个典型的 NFA 模拟问题。将 pattern 的每个位置看作 NFA 的状态：位置 i 表示"正准备匹配 pattern[i]"，`*` 产生 ε 转移（可跳过当前字符-星号对）。用"状态集"模拟法追踪所有可能的 pattern 位置。

## 完整代码

```python
def is_match(text: str, pattern: str) -> bool:
    """
    基于 NFA 状态集模拟的正则匹配

    为什么用 NFA 模拟而非递归回溯：状态集并行探索所有路径，
    避免回溯的指数爆炸，每个字符只处理一轮状态集
    """
    n = len(pattern)

    def epsilon_closure(states):
        """为什么需要闭包：X* 可匹配零次，需模拟从 X 跳到 X* 之后的 ε 转移"""
        stack = list(states)
        result = set(states)
        while stack:
            s = stack.pop()
            if s + 1 < n and pattern[s + 1] == '*':
                if s + 2 not in result:
                    result.add(s + 2)
                    stack.append(s + 2)
        return result

    states = epsilon_closure({0})  # 初始状态集 = 位置 0 + ε 跳过

    for ch in text:
        next_states = set()
        for s in states:
            if s >= n:
                continue
            if s + 1 < n and pattern[s + 1] == '*':  # 后跟 '*'
                if pattern[s] == ch or pattern[s] == '.':
                    next_states.add(s)  # 匹配后留在原地（* 可继续匹配更多）
                # 匹配零次：ε 闭包会处理跳到 s + 2
            elif pattern[s] == ch or pattern[s] == '.':
                next_states.add(s + 1)  # 普通匹配，前进一个字符
        states = epsilon_closure(next_states)
        if not states:
            return False

    return n in states  # 是否存在一条路径走到了 pattern 末尾


def test():
    cases = [
        ("aa", "a", False),
        ("aa", "a*", True),
        ("ab", ".*", True),
        ("aab", "c*a*b", True),
        ("", "a*", True),
        ("", "a", False),
        ("abc", "a.c", True),
        ("aaa", "ab*", False),
        ("a", "ab*", True),
    ]
    for text, pat, expected in cases:
        result = is_match(text, pat)
        mark = "✓" if result == expected else "✗"
        print(f'{mark} is_match("{text}", "{pat}") = {result}')


if __name__ == "__main__":
    test()
```
