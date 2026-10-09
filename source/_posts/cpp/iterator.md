---
title: 手写二叉搜索树迭代器：中序遍历的底层逻辑
date: 2021-08-07 19:33:30
updated: 2026-10-09
author: ws
description: 通过 SGI STL 风格的树迭代器，理解 ++/-- 如何在 O(1) 均摊时间内完成中序遍历
categories: ["C++"]
tags: ["C++", "数据结构", "迭代器"]
cover:
---

## 引言

`std::map`/`std::set` 的迭代器是双向的：`++it` 一步跳到中序后继，`--it` 一步跳到中序前驱，过程中不需要任何额外的栈。这套机制的核心是「每个节点多存一个父指针 + 一个 header 哨兵」。本文按 SGI STL 的思路实现二叉搜索树 `ws::bst`，把 `++`/`--` 的每个分支用图示讲透，最后给出可直接编译运行的完整代码，并整理出几种常见的错误写法与推荐实现。全部代码在 C++17 + ASan/UBSan 下实测通过。

## 中序遍历的三种实现

| 方式 | 额外内存 | 是否支持 `--` | 特点 |
| --- | --- | --- | --- |
| 递归函数 | O(h) 调用栈 | 不方便 | 代码最短，深树可能爆栈 |
| 显式栈 | O(h) 堆内存 | 很困难 | 每个迭代器要复制一份栈 |
| 父指针迭代器 | 每节点 1 个指针 | 支持 | 迭代器只存一个节点指针，双向移动 |

表中 h 是树高。第三种是 SGI STL 以及 libstdc++、libc++ 的 map/set 采用的方案：节点里多一个 `_M_parent`，换来迭代器状态下只有一个节点指针，复制、比较、`--` 都极其廉价。

复杂度上，单次 `++` 最坏沿父链走 O(h)，但把整棵树中序遍历一遍，每条边最多被「从子指向父」经过一次，n 次自增总计 O(n)，也就是**均摊 O(1)**。

## 节点设计与 header 哨兵

节点需要 `_M_parent`、`_M_left`、`_M_right` 三个指针。这里先把指针抽到一个不含数据的基类 `_Node_base` 里——所有树算法（找最左、找最右、沿父链走）都只操作基类指针，只有真正取值时才 `static_cast` 回带模板参数的 `_Node<T>`。这样插入、维护 header、遍历等逻辑都不必知道 T 是什么：

```cpp
#ifndef WS_BST_HPP
#define WS_BST_HPP

#include <cstddef>
#include <iterator>
#include <type_traits>
#include <utility>

namespace ws {

// 所有节点共有的头部：算法只在三个指针上工作，取值时再 static_cast 到 _Node<T>。
struct _Node_base {
    _Node_base* _M_parent = nullptr;
    _Node_base* _M_left = nullptr;
    _Node_base* _M_right = nullptr;
};

template <typename T>
struct _Node : _Node_base {
    T _M_value;
    explicit _Node(const T& value) : _M_value(value) {}
};
```

再看 header（哨兵）。它本身也是一个节点，但不存数据，三条指针的含义如下（以依次插入 8、3、10、1、6、14、4、7、13 的树为例）：

```text
   _M_header（哨兵，同时充当 end()）
     _M_parent ------> nullptr          ← 识别 header 的标记
     _M_left --------> 1                ← 整棵树的最左节点
     _M_right -------> 14               ← 整棵树的最右节点

             8   ← root（存在 bst::_M_root，8->_M_parent = &_M_header）
            / \
           3   10
          / \    \
         1   6    14
            / \   /
           4   7 13
```

由此确定几条不变量：

- `end()` 就是指向 `_M_header` 的迭代器；空树时 `header._M_left` 和 `header._M_right` 都指向自己，所以 `begin() == end()`；
- 根节点的 `_M_parent` 指向 header，其余节点指向真实父节点；
- `header._M_parent` **恒为 `nullptr`**，这是 `++`/`--` 里区分「普通节点」和「到达末尾」的唯一标记。

这里与 libstdc++ 的实现有一个刻意的差异：它把树根存在 `header._M_parent` 里，靠红黑树的颜色位（`_M_color == red && _M_parent->_M_parent == _M_node`）来识别 header。本实现没有颜色位，如果照搬「parent 存 root」，这个判断就不成立。为了让 `++`/`--` 的判断显式、无魔法，root 单独放在 `bst::_M_root` 里，header 的 parent 永远为空，代价只是树对象多存一个指针。

## 迭代器骨架：traits、构造与 const 转换

先说类型设计。`_Iterator` 是一个三参数的类模板 `_Iterator<T, Ref, Ptr>`：`iterator` 是 `_Iterator<T, T&, T*>`，`const_iterator` 是 `_Iterator<T, const T&, const T*>`。**节点指针本身不区分 const，值的可写性由 Ref/Ptr 表达**——这是实现时最容易走偏的地方，后面会展开。迭代器自身标成 `std::bidirectional_iterator_tag`，`std::iterator_traits` 就能识别它，标准算法和范围 for 都能直接使用：

```cpp
template <typename T, typename Ref, typename Ptr>
class _Iterator {
public:
    // SGI STL 惯例：_M_ 前缀 = 内部成员。比较函数直接读它，
    // 省掉了一大串 friend 声明。
    _Node_base* _M_node = nullptr;

    using iterator_category = std::bidirectional_iterator_tag;
    using value_type = T;
    using difference_type = std::ptrdiff_t;
    using pointer = Ptr;
    using reference = Ref;

    _Iterator() noexcept = default;
    explicit _Iterator(_Node_base* node) noexcept : _M_node(node) {}

    // 隐式转换 iterator -> const_iterator；反向转换被禁用。
    template <typename R2, typename P2,
              typename = std::enable_if_t<std::is_convertible<P2, Ptr>::value>>
    _Iterator(const _Iterator<T, R2, P2>& other) noexcept : _M_node(other._M_node) {}

    reference operator*() const noexcept {
        return static_cast<_Node<T>*>(_M_node)->_M_value;
    }
    pointer operator->() const noexcept { return &operator*(); }

    _Iterator& operator++() noexcept {
        _M_increment();
        return *this;
    }
    _Iterator operator++(int) noexcept {
        _Iterator tmp(*this);
        _M_increment();
        return tmp;
    }
    _Iterator& operator--() noexcept {
        _M_decrement();
        return *this;
    }
    _Iterator operator--(int) noexcept {
        _Iterator tmp(*this);
        _M_decrement();
        return tmp;
    }

private:
    // 下面两个函数实现 ++/-- 的核心逻辑
```

几个细节：前置 `++`/`--` 返回引用，后置返回旧值拷贝；转换构造用 `enable_if` 限定「目标 Ptr 可以由源 Ptr 隐式转换」，也就是只放行 `iterator → const_iterator`，反向编译不过；`_M_node` 直接用 `_Node_base*` 且放在 `public`，比较运算符模板可以直接读它，省掉了一大串 friend 声明——STL 实现里 `_M_` 前缀本来就表示「内部使用，后果自负」。

## `++it`：中序后继的两种情况

中序后继只有两种可能，取决于当前节点有没有右子树。

**情况一：有右子树。** 中序顺序是「左、根、右」，所以当前节点之后、进入父节点序列之前，会先走完它的右子树，后继就是右子树的**最左**节点。下图里 3 有右子树 {6, 4, 7}，中序是 3、4、6、7，故 `++` 从 3 走到 4：

```text
      3  ← 当前节点，有右子树
       \
        6
       /
      4      ← 右子树最左节点 = 3 的后继
```

**情况二：没有右子树。** 此时该子树已经走完，需要沿父指针向上，直到「当前节点是父节点的左孩子」为止——那个父节点就是后继。下图的中序是 6、7、8，从 7 出发：7 是 6 的右孩子，继续向上；6 是 3 的右孩子，继续向上；3 是 8 的左孩子，停下，后继是 8：

```text
        8      ← 后继：第一个「当前节点是其左孩子」的父节点
       /
      3
       \
        6
         \
          7    ← 从 7 出发，沿 parent 向上
```

**走到末尾。** 对最大节点 14 做 `++`：14 是 10 的右孩子 → 10 是 8 的右孩子 → 8 的父节点是 header。此时拿 8 和 `header._M_right`（也就是 14）比较，不相等，循环结束，落到 header，即 `end()`。如果根节点恰好就是最右节点（根没有右子树），循环会在把 `_M_node` 移到 header 之后再检查一次父节点，发现 `p == nullptr` 直接退出，`if (p)` 保证不会把迭代器再挪回根。

## `--it`：对称的两种情况与 `--end()`

`--` 完全对称。**情况一：有左子树。** 前驱是左子树的**最右**节点。下图中序是 3、4、6、7、8，8 的前驱是左子树中的最大节点 7，`--` 从 8 走到 7：

```text
        8      ← 当前节点，有左子树
       /
      3
       \
        6
       / \
      4   7    ← 左子树最右节点 7 = 8 的前驱
```

**情况二：没有左子树。** 沿父指针向上，直到「当前节点是父节点的右孩子」，那个父节点就是前驱。还是上图，从 4 出发：4 是 6 的左孩子，向上到 6；6 是 3 的右孩子，停下，前驱是 3（中序里 3 紧挨着 4）。

**`--end()` 的特例。** `end()` 指向 header，它的 `_M_parent` 是 `nullptr`。所以 `_M_decrement` 的第一个分支直接取 `header._M_right`，一步回到最右节点（也就是全树最大值）。这解释了为什么 header 必须维护 `_M_right`。

顺带一提，对最小节点做 `--`（相当于 `--begin()`）在标准里是未定义行为；本实现会安静地绕到 header，也就是得到 `end()`，但不要依赖这个行为。

两个函数的完整实现如下，循环里的 `p &&`、`p->_M_parent &&` 都是为「即将抵达 header」准备的护栏：

```cpp
    void _M_increment() noexcept {
        if (_M_node->_M_right) {
            // 情况一：右子树存在，后继 = 右子树的最左节点
            _M_node = _M_node->_M_right;
            while (_M_node->_M_left) _M_node = _M_node->_M_left;
        } else {
            // 情况二：沿父指针向上，直到当前节点是父节点的左孩子
            _Node_base* p = _M_node->_M_parent;
            while (p && _M_node == p->_M_right) {
                _M_node = p;
                p = p->_M_parent;
            }
            if (p) _M_node = p;   // p 为空说明已经走到 header（end()）
        }
    }

    void _M_decrement() noexcept {
        if (_M_node->_M_parent == nullptr) {
            // header 的 parent 恒为空：--end() 回到最右节点
            _M_node = _M_node->_M_right;
        } else if (_M_node->_M_left) {
            // 情况一：左子树存在，前驱 = 左子树的最右节点
            _M_node = _M_node->_M_left;
            while (_M_node->_M_right) _M_node = _M_node->_M_right;
        } else {
            // 情况二：沿父指针向上，直到当前节点是父节点的右孩子
            _Node_base* p = _M_node->_M_parent;
            while (p->_M_parent && _M_node == p->_M_left) {
                _M_node = p;
                p = p->_M_parent;
            }
            _M_node = p;
        }
    }
};
```

`--` 的情况二循环条件用 `p->_M_parent &&`：当走到 header 时，`header._M_parent` 为 `nullptr`，循环停止，最后的 `_M_node = p` 正好把迭代器落在 header 上，等价于 `end()`。

## 混合比较：iterator 与 const_iterator

二者是同一个类模板的不同实例。比较不必写成一堆重载：只要一个带两组 Ref/Ptr 参数的函数模板，`iterator`、`const_iterator` 以及它们之间的任意组合都用这一个实现，语义都是「内部节点指针是否相同」。这就是「不需要 friend」的好处——`_M_node` 是公有内部成员，模板直接读：

```cpp
template <typename T, typename R1, typename P1, typename R2, typename P2>
inline bool operator==(const _Iterator<T, R1, P1>& lhs,
                       const _Iterator<T, R2, P2>& rhs) noexcept {
    return lhs._M_node == rhs._M_node;
}

template <typename T, typename R1, typename P1, typename R2, typename P2>
inline bool operator!=(const _Iterator<T, R1, P1>& lhs,
                       const _Iterator<T, R2, P2>& rhs) noexcept {
    return !(lhs == rhs);
}
```

## 树对象：header 维护与 insert

最后是把它们组装起来的 `bst`。`insert` 沿普通 BST 规则下行，同时做三件事：记录父节点、更新 `header._M_left/_M_right`（最左/最右）、维护 `_M_count`。等值元素直接返回 `{已有迭代器, false}`。`clear` 后序遍历回收全部节点，并把 header 重置回空树状态：

```cpp
template <typename T>
class bst {
public:
    using iterator = _Iterator<T, T&, T*>;
    using const_iterator = _Iterator<T, const T&, const T*>;

private:
    _Node_base _M_header;      // 哨兵：parent 恒为 nullptr，left/right = 最左/最右
    _Node_base* _M_root = nullptr;
    std::size_t _M_count = 0;

    void _M_reset() noexcept {
        _M_header._M_parent = nullptr;
        _M_header._M_left = &_M_header;
        _M_header._M_right = &_M_header;
        _M_root = nullptr;
    }

    static void _M_destroy(_Node_base* node) noexcept {
        if (!node) return;
        _M_destroy(node->_M_left);
        _M_destroy(node->_M_right);
        delete static_cast<_Node<T>*>(node);
    }

public:
    bst() noexcept { _M_reset(); }
    ~bst() { clear(); }

    bool empty() const noexcept { return _M_count == 0; }
    std::size_t size() const noexcept { return _M_count; }

    iterator begin() noexcept { return iterator(_M_header._M_left); }
    iterator end() noexcept { return iterator(&_M_header); }
    // _M_header._M_left 在 const 方法里读出来仍是非 const 指针，可直接用；
    // &_M_header 则会变成 const _Node_base*，这里显式去掉 const（见下文讲解）。
    const_iterator cbegin() const noexcept {
        return const_iterator(_M_header._M_left);
    }
    const_iterator cend() const noexcept {
        return const_iterator(const_cast<_Node_base*>(&_M_header));
    }

    // 返回 {指向该值的迭代器, 是否真的插入了}；重复值不插入。
    std::pair<iterator, bool> insert(const T& value) {
        _Node_base* parent = &_M_header;
        _Node_base* cur = _M_root;
        while (cur) {
            parent = cur;
            T& v = static_cast<_Node<T>*>(cur)->_M_value;
            if (value < v)
                cur = cur->_M_left;
            else if (v < value)
                cur = cur->_M_right;
            else
                return {iterator(cur), false};
        }

        _Node<T>* node = new _Node<T>(value);
        node->_M_parent = parent;
        if (parent == &_M_header)
            _M_root = node;                          // 空树：新节点 = 根
        else if (value < static_cast<_Node<T>*>(parent)->_M_value)
            parent->_M_left = node;
        else
            parent->_M_right = node;

        // 维护 header 的最左/最右指针
        if (_M_header._M_left == &_M_header ||
            value < static_cast<_Node<T>*>(_M_header._M_left)->_M_value)
            _M_header._M_left = node;
        if (_M_header._M_right == &_M_header ||
            static_cast<_Node<T>*>(_M_header._M_right)->_M_value < value)
            _M_header._M_right = node;

        ++_M_count;
        return {iterator(node), true};
    }

    iterator find(const T& value) noexcept {
        _Node_base* cur = _M_root;
        while (cur) {
            T& v = static_cast<_Node<T>*>(cur)->_M_value;
            if (value < v)
                cur = cur->_M_left;
            else if (v < value)
                cur = cur->_M_right;
            else
                return iterator(cur);
        }
        return end();
    }

    void clear() noexcept {
        _M_destroy(_M_root);
        _M_reset();
        _M_count = 0;
    }
};

} // namespace ws

#endif // WS_BST_HPP
```

`cend()` 里那行 `const_cast` 值得解释。迭代器内部统一存非 const 的 `_Node_base*`（因为 `++`/`--` 要改写它），而在 `const` 成员函数里，`&_M_header` 的类型是 `const _Node_base*`，所以必须显式去掉 const。而 `_M_header._M_left` 不同：它是「const 指针对象的**值拷贝**」，读出来仍是非 const 的 `_Node_base*`，直接可用——这一点很容易想当然。值本身的 const 性由 `Ref`/`Ptr` 控制，`const_iterator` 的 `operator*` 返回 `const T&`，不会泄露写权限。

## 常见错误写法与推荐实现

先看节点指针的类型。一种常见的错误写法是把 `_M_node` 存成 `_Base_const_ptr`（指向 const 节点的指针），看起来「更 const」，实际只是把不可写性加在了节点上：迭代器沿树移动、把节点交给以 `_Base_ptr` 为参数的辅助函数时都得 `const_cast` 绕回来，而真正需要控制的「值能否修改」反倒由 `_Ref` 单独表达。正确的划分是：**节点指针只是内部实现细节，不加 const；「值能否修改」交给 Ref/Ptr 模板参数**。其余几种常见的错误写法：

1. **在头文件里 `#define NULL 0`。** 这种宏会污染所有包含该头文件的翻译单元，还会和 `<cstddef>` 里的 `NULL` 冲突。C++11 起直接用 `nullptr`。
2. **不要把 root 存进 `header._M_parent`。** 上面的 `_M_increment`/`_M_decrement` 用 `!_M_node->_M_parent` 作为「到达 header」的判据，这要求 header 的 parent 为空；若照搬 SGI 的约定（parent 存 root），遍历到末尾时会走错：增量循环会把 `end()` 再挪回根节点，而不是停在 header。本文明确约定 parent 恒空、root 另存。
3. **`operator++()` 返回 `_Self` 值拷贝。** 前置自增应返回 `_Self&`，标准库算法依赖这一点（否则 `*++it = x` 这类写法作用在临时对象上）。
4. **六个 friend 声明 + 六个比较重载太重。** 朴素写法会给 `==`/`!=` 各写三个重载（普通、const/non-const 两种混搭），漏一个就编译不过、且极易写错模板参数。换成两个通用函数模板，顺便获得任意组合的比较能力。
5. **转换构造的写法值得推敲。** 写成 `_Iterator(iterator const& __THAT)`，对 const 实例而言它扮演转换构造，对 non-const 实例而言又与同类拷贝语义重叠；显式的 `enable_if` 模板转换构造意图更清楚：只允许 const 性增强。
6. **`protected` 继承 `_Base_iterator` 没有必要。** 多一层间接，而且外部无法把迭代器当基类使用，收益为零；指针和移动逻辑直接放进 `_Iterator` 即可。
7. **`--end()` 依赖 `_M_decrement` 的 header 分支，而这个分支成立的前提是 header 的 parent 为空。** 分支里写的是 `if (!_M_node->_M_parent) { _M_node = _M_node->_M_right; }`，逻辑本身对，但它和第 2 条的约定是一体的：没有「parent 恒空、root 另存」这个前提，`--end()` 和 `++` 收尾都会落到错误位置。本文把这条不变量写在最前面，并用 `--end()=14` 的用例验证。

## 编译与运行

测试覆盖：中序遍历是否有序、`++` 到 `end()` 再 `--` 回到 `begin()`、`--end()`、求和、`find`、重复插入、`iterator` 与 `const_iterator` 混用、`clear` 后复用。整棵树按上面的节点定义和 header 约定组装：

```cpp
#include "ws_bst.hpp"

#include <algorithm>
#include <iostream>
#include <numeric>
#include <string>
#include <type_traits>
#include <vector>

template <typename Tree>
void print_inorder(const std::string& label, const Tree& tree) {
    std::cout << label << " (size=" << tree.size() << "):";
    for (auto it = tree.cbegin(); it != tree.cend(); ++it)
        std::cout << ' ' << *it;
    std::cout << '\n';
}

int main() {
    using Tree = ws::bst<int>;

    static_assert(std::is_same_v<
        std::iterator_traits<Tree::iterator>::iterator_category,
        std::bidirectional_iterator_tag>);
    static_assert(std::is_same_v<
        std::iterator_traits<Tree::const_iterator>::value_type, int>);

    std::cout << std::boolalpha;

    Tree t;
    for (int v : {8, 3, 10, 1, 6, 14, 4, 7, 13}) t.insert(v);
    print_inorder("in-order", t);

    std::vector<int> values(t.cbegin(), t.cend());
    std::cout << "sorted=" << std::is_sorted(values.begin(), values.end())
              << " size=" << t.size() << '\n';

    // ++ 走到 end()，再 -- 原路回到 begin()
    auto it = t.begin();
    while (it != t.end()) ++it;
    while (it != t.begin()) --it;
    std::cout << "++ then -- back to begin: " << (it == t.begin()) << '\n';

    auto last = t.end();
    --last;                       // --end() 应回到最大元素
    std::cout << "--end()=" << *last << '\n';

    long long sum = std::accumulate(t.begin(), t.end(), 0LL);
    std::cout << "sum=" << sum << '\n';

    auto f = t.find(6);
    std::cout << "find(6)=" << (f != t.end() ? std::to_string(*f) : "not found")
              << " find(99)=" << (t.find(99) == t.end() ? "end" : "found") << '\n';

    auto [pos, ok] = t.insert(6);   // 重复值不插入
    std::cout << "insert duplicate 6: ok=" << ok
              << " *pos=" << *pos << " size=" << t.size() << '\n';

    // iterator 与 const_iterator 可以互相比较、双向隐式转换
    Tree::const_iterator ci = t.begin();
    std::cout << "iterator == const_iterator: " << (t.begin() == ci) << '\n';
    const Tree& ct = t;
    print_inorder("const iteration", ct);

    t.clear();
    std::cout << "after clear: size=" << t.size()
              << " begin()==end(): " << (t.begin() == t.end()) << '\n';
    t.insert(42);
    print_inorder("reused", t);
}
```

编译运行（`-Wall -Wextra` 零警告，ASan/UBSan 无报错，退出码 0）：

```bash
clang++ -std=c++17 -Wall -Wextra -fsanitize=address,undefined -g -o ws_bst_test main.cpp
./ws_bst_test
```

```text
in-order (size=9): 1 3 4 6 7 8 10 13 14
sorted=true size=9
++ then -- back to begin: true
--end()=14
sum=66
find(6)=6 find(99)=end
insert duplicate 6: ok=false *pos=6 size=9
iterator == const_iterator: true
const iteration (size=9): 1 3 4 6 7 8 10 13 14
after clear: size=0 begin()==end(): true
reused (size=1): 42
```

中序遍历输出严格递增，`sorted=true` 说明迭代器走的就是正确的中序；`++` 到 `end()` 后再 `--` 回到 `begin()`、`--end()=14` 验证了两侧边界；`sum=66`（1+3+4+6+7+8+10+13+14）说明迭代器可以直接喂给标准算法。

## 总结

- `++`/`--` 各只有两种情况：有子树时走到「右子树最左」/「左子树最右」，没有子树时沿父指针向上找「第一个左/右拐点」；每步最坏 O(h)，整棵树遍历一遍均摊 O(1)。
- header 哨兵同时承担三重角色：`end()` 的落点、最左/最右节点的记录者、以及 `--end()` 的跳板；它的 `_M_parent` 恒为 `nullptr` 是本实现识别「到达末尾」的标记。
- 迭代器用 Ref/Ptr 两个模板参数区分 `iterator`/`const_iterator`，节点指针保持非 const；转换构造限制为 const 性增强；比较用单个函数模板覆盖全部组合。
- 在头文件里 `#define NULL`、把 root 塞进 `header._M_parent`、前置自增返回临时值、六个 friend 重载等，都是实现这类迭代器时常见的错误写法；文中逐条给出推荐做法，并由编译测试覆盖。
