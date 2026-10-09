---
title: 从零实现 C++ 双向链表：哨兵节点与拷贝控制
date: 2021-08-08 13:30:25
updated: 2026-10-09
author: ws
description: 手写类似 std::list 的双向链表，讲清哨兵节点、迭代器与深拷贝
categories: ["C++"]
tags: ["C++", "数据结构", "list"]
cover:
---

## 引言

链表是「面试必写、工程少写」的典型数据结构。本文从零实现一个类似 `std::list` 的双向链表 `ws::list`，重点讲清三件事：带哨兵的循环结构如何消掉首尾操作的特殊判断、双向迭代器怎么和哨兵配合、以及拷贝控制为何必须深拷贝。本文代码在 C++17 下用 `clang++ -Wall -Wextra -fsanitize=address,undefined` 实测通过。适合用过 C++ 模板、想弄懂容器内部实现细节的读者。

自己动手写链表，真正的难点不在把指针接对，而在拷贝控制、维护计数和处理边界。文中「常见错误写法与正确做法」一节会先看几段看起来没问题、实际会出事的代码——浅拷贝、`insert` 漏计数、`clear` 之后对象报废等——再给出推荐写法，逐条说明原因。代码整体沿用 STL 风格：容器名 `ws::list`，节点名 `ListNode`。

## 先想清楚：什么时候才需要链表

选择容器先看操作组合。链表和动态数组的复杂度差异如下：

| 操作 | 双向链表 | 动态数组（vector） |
| --- | --- | --- |
| 按下标随机访问 | O(n) | O(1) |
| 头部插入/删除 | O(1) | O(n) |
| 尾部插入/删除 | O(1) | 摊还 O(1) |
| 已知迭代器位置的插入/删除 | O(1) | O(n) |
| 查找某个值 | O(n) | O(n)（有序时可二分） |
| 每个元素的额外内存 | 2 个指针（64 位下 16 字节） | 通常没有，扩容时才有冗余 |

但复杂度只是故事的一半。数组的元素在内存里连续排布，顺序遍历时一条 64 字节的 cache line 能装下多个元素；链表的节点由 `new` 逐个分配，地址分散在堆上，每前进一步都可能触发一次 cache miss，还要多做一次指针解引用。所以实践中 `std::list` 经常比 `std::vector` 慢一个数量级，即使插入删除的复杂度看起来更优。

一个常见的经验是：默认用 `vector`；只有下面这些情况才考虑链表——你**已经持有**要操作位置的迭代器且需要频繁在中间插入/删除、元素本身很大（拷贝/搬移成本高）、或者需要「插入/删除后其他元素的指针、迭代器保持稳定」这一保证。如果只是频繁在两端操作，`std::deque` 通常比 `list` 更合适。

## 设计：带哨兵的循环双向链表

先看结构。哨兵（sentinel）是一个不存数据的节点，它的存在让空链表和非空链表拥有完全相同的结构：

```text
   _M_header ⇄ [0] ⇄ [1] ⇄ [2] ⇄ [3] ──┐
       ▲                               │
       └───────────────────────────────┘
```

- `_M_header.next` 指向首元素，`_M_header.prev` 指向尾元素；
- 最后一个元素的 `next` 指回 `_M_header`，第一个元素的 `prev` 也指向 `_M_header`，整体是一个环；
- 空链表时 `next` 和 `prev` 都指向 `_M_header` 自己。

由此得到两个直接好处。第一，**首尾操作不再需要分支**：`push_front` 就是「在 `header.next` 之前插入」，`push_back` 就是「在 `header` 之前插入」，两者可以归约成同一个私有函数 `_M_insert_before`；删除首元素就是删除 `header.next`，删除尾元素就是删除 `header.prev`，也不用判断链表是否为空。第二，**`end()` 就是指向哨兵的迭代器**，`--end()` 沿着 `header.prev` 一步就能回到最后一个元素。

哨兵直接按值嵌入容器对象，而不是 `new` 出来：生命周期随对象自动管理，少一次动态分配，也不存在「忘了 delete 哨兵」的问题。

## 节点与迭代器

节点就是三个字段加一个构造函数。注意构造函数把三个成员都给了默认值，指针默认 `nullptr`，不再出现未初始化状态：

```cpp
#ifndef WS_LIST_HPP
#define WS_LIST_HPP

#include <cstddef>
#include <iterator>
#include <utility>

namespace ws {

template <typename T>
struct ListNode {
    T data;
    ListNode* prev;
    ListNode* next;

    explicit ListNode(const T& value = T(),
                      ListNode* p = nullptr,
                      ListNode* n = nullptr)
        : data(value), prev(p), next(n) {}
};
```

迭代器直接持有节点指针，`++`/`--` 就是沿 `next`/`prev` 走一步。`end()` 返回指向 `_M_header` 的迭代器，所以 `--end()` 会自动落到尾元素上——这正是哨兵结构送来的便利：

```cpp
template <typename T>
class list {
    using Node = ListNode<T>;

    Node _M_header;      // 哨兵：不存数据，next 指向首元素，prev 指向尾元素
    std::size_t _count;

public:
    using value_type = T;
    using size_type = std::size_t;

    class iterator {
        Node* _node;

        explicit iterator(Node* node) : _node(node) {}

        friend class list;   // 只允许 list 从裸节点构造迭代器

    public:
        using iterator_category = std::bidirectional_iterator_tag;
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using pointer = T*;
        using reference = T&;

        iterator() : _node(nullptr) {}

        reference operator*() const { return _node->data; }
        pointer operator->() const { return &_node->data; }

        iterator& operator++() {
            _node = _node->next;
            return *this;
        }
        iterator operator++(int) {
            iterator tmp(*this);
            ++(*this);
            return tmp;
        }

        iterator& operator--() {
            _node = _node->prev;
            return *this;
        }
        iterator operator--(int) {
            iterator tmp(*this);
            --(*this);
            return tmp;
        }

        bool operator==(const iterator& rhs) const { return _node == rhs._node; }
        bool operator!=(const iterator& rhs) const { return _node != rhs._node; }
    };
```

`iterator_category` 标成 `std::bidirectional_iterator_tag`，标准算法（`std::distance`、范围 for 等）就能认出这是个双向迭代器。构造函数私有、`friend class list`，是为了防止外部代码拿裸指针拼出非法迭代器。

## 内部积木：一切插入都归约为「在 pos 之前插」

先写几个私有辅助函数。`_M_init_header` 让哨兵自环；`_M_insert_before` 是所有插入的唯一入口，指针改写顺序是「先让新节点接上前后两个邻居，再让邻居改指新节点」，中间不存在指针悬空状态；`_M_erase` 摘除节点时先接好前后邻居再 `delete`：

```cpp
private:
    void _M_init_header() noexcept {
        _M_header.prev = &_M_header;
        _M_header.next = &_M_header;
    }

    void _M_insert_before(Node* pos, const T& value) {
        Node* node = new Node(value, pos->prev, pos);
        node->prev->next = node;
        node->next->prev = node;
        ++_count;
    }

    void _M_erase(Node* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
        delete node;
        --_count;
    }

    void _M_destroy() noexcept {
        Node* p = _M_header.next;
        while (p != &_M_header) {
            Node* next = p->next;
            delete p;
            p = next;
        }
    }
```

对外接口全部建立在这些辅助函数之上；但完整文件里它们的顺序在构造、拷贝控制之后。先把生命周期讲清楚，接口代码放在「对外的其余接口」一节，按文件顺序拼接即可。

## 拷贝控制：Rule of Five

链表是「拥有资源」的类（每个节点都要手动释放），所以析构、拷贝构造、拷贝赋值、移动构造、移动赋值五个函数都要自己写，这就是 Rule of Five。其中**两个拷贝操作必须是深拷贝**：如果只复制 `_M_header` 里的指针和 `_count`，两个对象会共享同一批节点，先析构的那一个把节点全部 `delete`，另一个再析构就是双重释放，而如果谁都不析构就是泄漏。

```cpp
public:
    list() : _count(0) { _M_init_header(); }

    list(const list& other) : _count(0) {
        _M_init_header();
        for (Node* p = other._M_header.next; p != &other._M_header; p = p->next)
            push_back(p->data);      // 逐个复制节点
    }

    list(list&& other) noexcept : _count(other._count) {
        if (other.empty()) {
            _M_init_header();
            return;
        }
        _M_header.next = other._M_header.next;   // 直接接管整条链，O(1)
        _M_header.prev = other._M_header.prev;
        _M_header.next->prev = &_M_header;
        _M_header.prev->next = &_M_header;
        other._M_init_header();                  // 被移动对象恢复为空链表
        other._count = 0;
    }

    ~list() { _M_destroy(); }
```

移动构造不复制任何节点，只把对方整条链「搬」过来，再把对方的哨兵重置成空链表，因此是 O(1) 且不分配内存，可以标 `noexcept`。被移动的对象处于合法状态，测试里会验证它之后还能继续使用。

拷贝赋值用经典的 **copy-and-swap**：先拷贝出一份临时对象（这一步可能抛异常，此时 `*this` 完全没被碰过），再和 `*this` 交换。交换只动指针和计数，O(1) 且不抛异常；临时对象在函数结束时析构，顺带释放了 `*this` 此前持有的节点：

```cpp
    list& operator=(const list& other) {
        if (this != &other) {
            list tmp(other);   // 先深拷贝，抛异常时 *this 不受影响
            swap(tmp);
        }
        return *this;
    }

    list& operator=(list&& other) noexcept {
        if (this != &other) {
            list tmp(std::move(other));
            swap(tmp);
        }
        return *this;
    }

    void swap(list& other) noexcept {
        if (this == &other) return;
        std::swap(_M_header.next, other._M_header.next);
        std::swap(_M_header.prev, other._M_header.prev);
        std::swap(_count, other._count);

        // 交换后如果“邻居”是对方的 header，说明对方原来是空链表，
        // 把自己接回自环即可；否则重设首尾节点的反向指针。
        if (_M_header.next == &other._M_header) {
            _M_header.next = _M_header.prev = &_M_header;
        } else {
            _M_header.next->prev = &_M_header;
            _M_header.prev->next = &_M_header;
        }

        if (other._M_header.next == &_M_header) {
            other._M_header.next = other._M_header.prev = &other._M_header;
        } else {
            other._M_header.next->prev = &other._M_header;
            other._M_header.prev->next = &other._M_header;
        }
    }
```

`swap` 里这段空链表判断是一个容易被忽略的边界：空链表的 `header.next` 指向**自己**，如果无脑交换后再按常规逻辑接回邻居，另一方的 header 会被串进环里，链表静默损坏。另一个容易忽略的细节是自赋值：`operator=` 里 `this != &other` 的判断就是为此存在的。

## 对外的其余接口

基于这两个函数，对外的接口全部是一两行的事。注意 `pop_back` 在空链表上会操作哨兵，和 `std::list` 一样属于未定义行为，调用方必须先 `empty()` 检查；`front`/`back` 同理：

```cpp
    iterator begin() { return iterator(_M_header.next); }
    iterator end() { return iterator(&_M_header); }

    bool empty() const { return _count == 0; }
    size_type size() const { return _count; }

    T& front() { return _M_header.next->data; }
    const T& front() const { return _M_header.next->data; }
    T& back() { return _M_header.prev->data; }
    const T& back() const { return _M_header.prev->data; }

    void push_front(const T& value) { _M_insert_before(_M_header.next, value); }
    void push_back(const T& value) { _M_insert_before(&_M_header, value); }

    void pop_front() { _M_erase(_M_header.next); }
    void pop_back() { _M_erase(_M_header.prev); }

    iterator insert(iterator pos, const T& value) {
        _M_insert_before(pos._node, value);
        return iterator(pos._node->prev);   // 返回新插入元素的迭代器
    }

    iterator erase(iterator pos) {
        Node* next = pos._node->next;
        _M_erase(pos._node);
        return iterator(next);              // 返回被删元素的下一个位置
    }

    bool contains(const T& value) const {
        for (Node* p = _M_header.next; p != &_M_header; p = p->next)
            if (p->data == value) return true;
        return false;
    }

    void clear() noexcept {
        _M_destroy();
        _M_init_header();   // 只清空元素，哨兵留着，对象继续可用
        _count = 0;
    }
};

} // namespace ws

#endif // WS_LIST_HPP
```

`clear()` 只删除元素节点并让哨兵重新自环，**不删除哨兵**，所以清空之后对象仍然可以正常 `push_back`。`insert`/`erase` 遵循标准库惯例返回迭代器：`insert` 返回新元素位置，`erase` 返回后继位置，这样连续删除可以写成 `it = l.erase(it)`。

## 常见错误写法与正确做法

不假思索写出来的链表，问题往往集中在下面这几处。每一条先给出常见的错误写法，再说明它错在哪、推荐怎么写：

1. **拷贝赋值写成浅拷贝。** 一种常见的错误写法是直接 `_pHead = other._pHead;`：两个对象共享同一批节点，连带先 `new` 出来的哨兵也被覆盖丢失。后果是双重释放加内存泄漏。正确写法：copy-and-swap 深拷贝。
2. **`swap` 借助拷贝构造和赋值实现。** 写成 `ws_list tmp = *this; *this = other; other = tmp;` 会依赖拷贝控制本身完全正确；一旦赋值运算符也调用 `swap`，两者会直接无限递归。正确写法：只交换指针和计数，O(1)。
3. **`insert` 忘记 `++_count`。** 插入后 `size()` 偏小，后续 `_GetNode` 按索引找节点时会漏掉尾部，`pop_back` 也删错位置。正确写法：插入统一从 `_M_insert_before` 走，计数在那里更新。
4. **`_GetNode` 里 `size_t pos;` 未初始化就参与 `while (pos++ < index)`。** 读未初始化变量是未定义行为，debug 下可能碰巧可用，优化后结果不可预测。另外用下标遍历找第 n 个元素，本身就是 O(n) 的额外成本。
5. **`clear()` 把哨兵也删了并置 `_pHead = NULL`。** 对象此后任何操作都会解引用空指针；析构函数若也调用 `clear()`，只是因为 `delete NULL` 恰好合法才侥幸通过。正确写法：`clear()` 只删元素、哨兵自环，对象保持可用。
6. **空链表上的 `pop`/`erase` 越界。** 像 `pop_back` 里写 `erase(_count - 1)`，`_count` 是无符号数，空链表时下溢成 `SIZE_MAX`；`assert(index >= 0 && index < _count)` 对 `size_t` 而言前半句恒真，且在 NDEBUG 下被整个去掉。正确写法：接口保证语义合法，空容器上的 `pop`/`front`/`back` 与标准库一致属于未定义行为，由调用方检查。
7. **`DNode(){}` 不初始化 `prev`/`next`。** 构造函数体为空，两个指针是垃圾值；`push_front` 在「链表非空」分支里只改了既有首元素的 `prev`，却没有把 `pNode` 接到 `_pHead->next` 上，导致新元素根本没被挂进链表。正确写法：指针成员一律 `nullptr` 初始化，插入走统一路径。
8. **`contain(ValueT& value)` 只接受左值。** 无法写 `contain(3)` 这样的常量或临时值。正确写法：参数改为 `const T&`，顺手把名字改成和标准库风格更接近的 `contains`。

## 复杂度与使用建议

| 操作 | ws::list / std::list | std::vector |
| --- | --- | --- |
| `push_front` | O(1) | O(n) |
| `push_back` | O(1) | 摊还 O(1) |
| `pop_front` | O(1) | O(n) |
| `insert`/`erase`（已持迭代器） | O(1) | O(n) |
| 随机访问 | 不支持，需 O(n) 遍历 | O(1) |
| `clear` | O(n) | O(n) |

落到具体选择上：绝大多数场景用 `vector`，需要频繁头尾操作时先看 `deque`，只有确实需要在任意位置大量插删、或者依赖「插删后其他迭代器不失效」时才用 `list`。另外 `list` 每个元素额外背两个指针，遍历时的 cache miss 也最多，小对象场景下几乎没有赢面。

## 完整代码与运行验证

把上面各段按顺序拼起来就是完整的 `ws_list.hpp`（拼接后 206 行）。下面是测试程序，覆盖了基本操作、迭代器、深拷贝、移动、自赋值、空链表 swap 等边界：

```cpp
#include "ws_list.hpp"

#include <iostream>
#include <string>
#include <utility>

template <typename T>
void print_list(const std::string& label, ws::list<T>& l) {
    std::cout << label << " (size=" << l.size() << "):";
    for (auto& v : l) std::cout << ' ' << v;
    std::cout << '\n';
}

int main() {
    std::cout << std::boolalpha;

    ws::list<int> a;
    a.push_back(2);
    a.push_back(3);
    a.push_back(4);
    a.push_front(1);
    a.push_front(0);
    print_list("a", a);

    std::cout << "front=" << a.front() << " back=" << a.back() << '\n';
    std::cout << "contains(3)=" << a.contains(3)
              << " contains(9)=" << a.contains(9) << '\n';

    auto it = a.begin();
    ++it;
    ++it;                       // 指向元素 2
    a.insert(it, 99);           // 在 2 前面插入 99
    print_list("insert 99 before 2", a);

    it = a.erase(it);           // 删除 2，返回下一个元素的迭代器
    print_list("erase 2", a);
    std::cout << "*it after erase=" << *it << '\n';

    a.pop_front();
    a.pop_back();
    print_list("pop_front + pop_back", a);

    auto last = a.end();
    --last;                     // --end() 回到最后一个元素
    std::cout << "--end()=" << *last << '\n';

    // 深拷贝：两份数据应位于不同地址
    ws::list<int> b = a;
    b.push_back(777);
    std::cout << "a/b front addresses: " << static_cast<const void*>(&a.front())
              << " / " << static_cast<const void*>(&b.front()) << '\n';
    print_list("a", a);
    print_list("b", b);

    ws::list<int> c;
    c.push_back(-1);
    c = a;                      // 拷贝赋值
    print_list("c = a", c);

    ws::list<int>& cref = c;
    c = cref;                   // 自赋值：不应出错
    print_list("c = c", c);

    ws::list<int> d = std::move(b);   // 移动：O(1) 窃取节点
    print_list("d = move(b)", d);
    print_list("b after move", b);

    b.push_back(42);            // 移动后的对象仍可正常使用
    print_list("b reused", b);

    a.clear();
    print_list("a after clear", a);
    a.push_back(5);             // clear 之后对象依旧可用
    print_list("a reused", a);

    ws::list<int> e1;
    ws::list<int> e2;
    e2.push_back(100);
    e2.push_back(200);
    e1.swap(e2);                // 空链表与非空链表交换
    print_list("e1 after swap", e1);
    print_list("e2 after swap", e2);

    ws::list<int> e3;
    e1.swap(e3);                // 非空链表与空链表交换
    print_list("e1 swapped with empty", e1);
    print_list("e3 after swap", e3);

    ws::list<int> empty_src;
    e3 = empty_src;             // 拷贝赋值：源为空
    print_list("e3 = empty list", e3);

    ws::list<std::string> s;
    s.push_back("hello");
    s.push_back("world");
    print_list("s", s);
}
```

编译并运行（ASan/UBSan 会顺带检查越界、泄漏和未定义行为）：

```bash
clang++ -std=c++17 -Wall -Wextra -fsanitize=address,undefined -g -o ws_list_test main.cpp
./ws_list_test
```

实测输出，编译零警告，退出码 0：

```text
a (size=5): 0 1 2 3 4
front=0 back=4
contains(3)=true contains(9)=false
insert 99 before 2 (size=6): 0 1 99 2 3 4
erase 2 (size=5): 0 1 99 3 4
*it after erase=3
pop_front + pop_back (size=3): 1 99 3
--end()=3
a/b front addresses: 0x603000001d20 / 0x603000001db0
a (size=3): 1 99 3
b (size=4): 1 99 3 777
c = a (size=3): 1 99 3
c = c (size=3): 1 99 3
d = move(b) (size=4): 1 99 3 777
b after move (size=0):
b reused (size=1): 42
a after clear (size=0):
a reused (size=1): 5
e1 after swap (size=2): 100 200
e2 after swap (size=0):
e1 swapped with empty (size=0):
e3 after swap (size=2): 100 200
e3 = empty list (size=0):
s (size=2): hello world
```

两个地址不同，证明 `b` 是独立的一份数据；`b after move` 为空、随后还能 `push_back`，证明被移动对象状态合法；空链表 swap 的两组用例验证了 `swap` 对空链表的处理。

## 总结

- 哨兵把「空链表」「首元素」「尾元素」三种情况统一成同一套指针操作，`push_front` 和 `push_back` 因此能归约成一个 `_M_insert_before`，`--end()` 也能直接落到尾元素。
- 迭代器就是「节点指针 + 双向移动」，`end()` 指向哨兵；返回迭代器的 `insert`/`erase` 是标准库惯例，方便连续操作。
- 拥有资源的类必须实现 Rule of Five；两个拷贝操作一定要深拷贝，赋值用 copy-and-swap 同时拿到强异常安全和 O(1) 交换；移动只是接管指针，O(1) 且被移动对象要恢复成合法的空状态。
- 链表真正的代价在缓存局部性上：除非持迭代器做中间插删或依赖迭代器稳定性，否则优先用 `vector`。
- 浅拷贝、漏计数、`clear` 删除哨兵、未初始化指针、空链表越界是手写链表最容易踩的坑；文中对每种错误写法都给出了对应的正确做法，并由 ASan/UBSan 测试覆盖。
