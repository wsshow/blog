---
title: 从零实现 C++ 动态数组（上）：内存管理与扩容策略
date: 2021-07-18 16:18:30
updated: 2026-10-09
author: ws
description: 手写一个类似 std::vector 的动态数组，讲透扩容、拷贝控制与移动语义
categories: ["C++"]
tags: ["C++", "数据结构", "vector"]
cover:
---

`std::vector` 大概是 C++ 里使用频率最高的容器，也是最适合用来理解 RAII、对象生命周期和异常安全的练手项目：它同时牵涉堆内存的申请与释放、构造与析构的分离、拷贝与移动的选择、扩容时元素搬移等几乎所有核心话题。这篇文章从零实现一个 `ws_vector`，代码以 C++17 为标准，全部通过 `-Wall -Wextra` 与 ASan/UBSan 验证。

系列分上下两篇：上篇讲内存管理与扩容，下篇给容器加上符合 STL 规范的迭代器，让它能配合 `std::sort`、`std::find`。文末的测试代码完整可复现，命令和真实输出一并给出。

## 动态数组 vs 链表：先想清楚要什么

「动态数组」和「链表」解决同一个问题的两个方向：如何存放数量在运行时才知道的一组元素。取舍决定了实现的形态：

| 维度 | 动态数组（vector） | 双向链表（list） |
| --- | --- | --- |
| 随机访问 | O(1)，纯指针运算 | O(n)，顺着指针走 |
| 尾部插入 / 删除 | 摊还 O(1) | O(1) |
| 中间插入 / 删除 | O(n)，要搬移元素 | O(1)（已知位置时） |
| 内存布局 | 一整块连续内存 | 每个节点单独分配 |
| 缓存友好度 | 高，预取器友好 | 低，每个节点一次 cache miss |
| 失效规则 | 扩容后几乎全部失效 | 只有被删节点的迭代器失效 |

连续内存的收益经常被低估：顺序遍历数组时预取器能提前拉取后面的 cache line，而链表每跳一个节点都可能触发 miss，两者时间复杂度同为 O(n)，实测性能常常相差数倍。所以工程上的默认选择永远是 vector，只有在中间频繁插入删除、元素很大或指针必须长期稳定的场景，才轮到 list / deque。

动态数组的难点不在「访问」，而在「扩容」和「对象生命周期管理」。下面开始动手。

## 三个成员与不变量

整个容器只有三个成员，先记住它们——「大小」和「容量」是两回事：

```cpp
pointer _value = nullptr;   // 指向缓冲区首地址
size_type _size = 0;        // 已构造的元素个数（逻辑大小）
size_type _cap = 0;         // 缓冲区能容纳的元素个数（容量）
```

```text
_value
  │
  ▼
  +------+------+------+------+------+------+------+------+
  | 10   | 20   | 30   |  ?   |  ?   |  ?   |  ?   |  ?   |
  +------+------+------+------+------+------+------+------+
  ^                     ^                                   ^
 begin()               begin()+_size()                      begin()+_capacity()
  |<---- 已构造的对象（_size = 3）---->|<-- 只有内存，没有对象 -->|
```

任何时候都必须满足三条不变量：

- `0 <= _size <= _cap`；
- `_cap > 0` 时 `_value` 指向一块能放下 `_cap` 个元素的内存，否则为 `nullptr`；
- `[_value, _value + _size)` 区间内都是**活对象**，`[_value + _size, _value + _cap)` 区间内只有内存，没有对象。

第三条最能体现这份实现的设计取向：把「分配内存」和「构造对象」彻底分开。一种直觉的写法是用 `new ValueT[_cap]` 分配缓冲区，但这条语句其实做了两件事：分配内存，并且把 `_cap` 个元素全部默认构造出来。它要求 `ValueT` 可默认构造，而且当 `_cap` 远大于 `_size` 时，大量槽位构造了又析构，纯属浪费。更稳妥的做法是把两件事拆开：用 `::operator new(n * sizeof(T))` 只要内存；用 placement new（`::new (p) T(...)`）在指定地址上构造对象；用完手动调 `p->~T()` 销毁，再统一 `::operator delete` 释放内存。后文的 `_allocate`、`_construct_at`、`_destroy_range` 就是干这三件事的。

## 扩容：为什么必须按倍数增长

`push_back` 在容量够时不搬移，容量不足时要申请更大的块、把旧元素搬过去。每次只加 1 个位置的策略，连续 push n 次的总搬运量是：

```text
1 + 2 + 3 + ... + (n - 1) ≈ n² / 2
```

总代价 O(n²)，单次平均 O(n)。换成翻倍策略，容量序列为 1, 2, 4, 8, ...，总搬运量是：

```text
1 + 2 + 4 + ... + 2^k < 2n        （2^k >= n）
```

把不到 2n 次搬运摊到 n 次 `push_back` 上，每次平均是常数——这就是**摊还 O(1)**，单次最坏仍是 O(n)（刚好触发扩容那一次）。这是动态数组能立足的根本。

为什么常见实现选 2 倍或 1.5 倍而不是别的倍数？因为倍数增长还有一个隐性约束：**旧块释放后能不能被复用**。

| 策略 | 增长序列（从 1 起） | 旧块复用 | 最坏内存占用 | 典型实现 |
| --- | --- | --- | --- | --- |
| 2 倍 | 1, 2, 4, 8, 16, ... | 不能：新块总比此前所有块之和还大 | 约 2n | libstdc++ |
| 1.5 倍 | 1, 2, 3, 4, 6, 9, 13, 19, ... | 几步之后能用上旧块 | 约 1.5n | MSVC |

2 倍增长时，第 k 次申请的新块是 2^k，而此前所有块之和是 2^k - 1，旧块永远塞不下新需求，分配器没法复用，峰值内存接近 2n。1.5 倍是「斐波那契式增长」：1 + 2 = 3、2 + 3 > 4……几步之后新申请的大小就不大于之前释放块之和，峰值约 1.5n；代价是扩容更频繁（log₁.₅ n 比 log₂ n 多出约 70%）。本文用最简单的 2 倍，知道取舍即可。

## 动手实现：一份完整的 ws_vector

下面按头文件从上到下的顺序拆解实现。**每个代码块都是最终 `ws_vector.h` 的一段，依次拼接就是完整文件**；测试用 `main.cpp` 和真实运行结果放在文末。整个实现只有 `ws_vector.h` 一个头文件：不使用 MSVC 专用的 `DEBUG_CHECK_MEMORY_LEAKS` 宏和 `#pragma warning`，也不依赖任何自定义头文件。

### 文件骨架与类型别名

```cpp
#ifndef WS_VECTOR_H
#define WS_VECTOR_H

// 从零实现 std::vector（上篇）：C++17 单文件头 ws_vector.h
#include <algorithm>
#include <cassert>
#include <cstddef>
#include <initializer_list>
#include <limits>
#include <stdexcept>
#include <utility>

template <typename ValueT>
class ws_vector {
public:
    using value_type      = ValueT;
    using size_type       = std::size_t;
    using difference_type = std::ptrdiff_t;
    using reference       = ValueT&;
    using const_reference = const ValueT&;
    using pointer         = ValueT*;
    using const_pointer   = const ValueT*;
```

这组别名照抄 STL 容器的命名习惯：普通调用者用不到，但下篇实现迭代器、让算法识别容器时，`value_type`、`reference` 这些名字会被 `std::iterator_traits` 取用。`size_type` 用无符号的 `std::size_t`，`difference_type` 用有符号的 `std::ptrdiff_t`——符号性很关键，`end() - begin()` 是可能为负的差值。

### 构造函数与析构函数

```cpp
    // ---------------- 默认构造 / 带 count 的构造 / 析构 ----------------
    ws_vector() noexcept = default;

    explicit ws_vector(size_type count) {
        _allocate_storage(count);
        try { for (; _size < count; ++_size) _construct_at(_value + _size); }
        catch (...) { _destroy_all_and_free(); throw; }
    }

    ws_vector(size_type count, const ValueT& value) {
        _allocate_storage(count);
        try { for (; _size < count; ++_size) _construct_at(_value + _size, value); }
        catch (...) { _destroy_all_and_free(); throw; }
    }

    ws_vector(std::initializer_list<ValueT> init) {
        _allocate_storage(init.size());
        try { for (const ValueT& v : init) { _construct_at(_value + _size, v); ++_size; } }
        catch (...) { _destroy_all_and_free(); throw; }
    }

    ~ws_vector() { _destroy_all_and_free(); }
```

构造函数里有三个固定要点：

1. **先要内存，再逐个构造对象**。`_allocate_storage` 只拿一块裸内存，`_construct_at` 才是构造；`_size` 只统计构造成功的元素，所以它天然等于「已构造前缀的长度」。
2. **失败时销毁已构造的部分、释放内存、原样抛出**。对象构造没完成时析构函数不会被调用，清理必须在这里手动做——这恰恰是手写容器构造函数最容易漏掉的一步。
3. **`explicit` 修饰单参数构造函数**，避免 `ws_vector<int> v = 5` 这种隐式转换。`initializer_list` 版本让你能写 `ws_vector<int> v{1, 2, 3}`。

构造函数依赖 `_allocate_storage`、`_construct_at`、`_destroy_all_and_free` 三个私有原语，它们的定义在文件末尾的 private 区，最后一个小节会展开。

### 容量查询

```cpp
    // ---------------- 容量 ----------------
    bool empty() const noexcept { return _size == 0; }
    size_type size() const noexcept { return _size; }
    size_type capacity() const noexcept { return _cap; }
    size_type max_size() const noexcept { return std::numeric_limits<size_type>::max() / sizeof(ValueT); }
```

`empty()` 判断的是 `_size == 0` 而不是 `_cap == 0`：`clear()` 之后 `empty()` 为真，但容量通常还留着。`max_size()` 按元素大小换算理论上限；如果图省事直接返回 `SIZE_MAX`，对 `int` 来说就夸大了 4 倍。

### push_back / emplace_back / pop_back

```cpp
    // ---------------- 尾部添加 / 删除 ----------------
    void push_back(const ValueT& value) {
        _grow_if_needed(_size + 1);
        _construct_at(_value + _size, value);
        ++_size;
    }

    void push_back(ValueT&& value) {
        _grow_if_needed(_size + 1);
        _construct_at(_value + _size, std::move(value));
        ++_size;
    }

    template <typename... Args>
    reference emplace_back(Args&&... args) {
        _grow_if_needed(_size + 1);
        _construct_at(_value + _size, std::forward<Args>(args)...);
        ++_size;
        return _value[_size - 1];
    }

    void pop_back() {
        assert(!empty());   // 空容器 pop 是 UB，和 std::vector 一致，debug 下先拦住
        --_size;
        (_value + _size)->~ValueT();
    }
```

`push_back` 的三行顺序有讲究：**先确保容量，再构造元素，最后才 `++_size`**。如果构造抛异常，`_size` 保持原值，容器状态与调用前一致；反过来先加 `_size`，抛异常后容器会认为自己拥有一个没构造成功的元素，析构时对垃圾地址调用 `~ValueT()`。这就是最基本的**强异常安全保证**：操作失败，容器不变。

两个重载分别接左值（拷贝构造）和右值（移动构造）；`emplace_back` 把参数直接转发给元素构造函数，省掉临时对象。

`pop_back` 在空容器上是未定义行为——和 `std::vector` 的约定一致，标准库也不检查。如果省掉检查直接 `--_size`，空容器上 0 会下溢成 `size_t` 最大值，之后所有基于 `size()` 的循环全部越界。我们用 assert 在 debug 构建里拦截，release 构建保持标准库同样的零开销语义。

### reserve 与 resize：一个改容量，一个改大小

这两个函数最容易用混，语义完全不同：

| 调用 | 改哪个成员 | 会不会构造元素 | 典型用途 |
| --- | --- | --- | --- |
| `reserve(n)` | 只保证 `_cap >= n` | 不会 | 预知元素数量，提前一次分配，避免多次扩容 |
| `resize(n)` | 保证 `_size == n` | 变大时构造，变小时销毁 | 按「已存在的元素个数」使用容器 |

```cpp
    // ---------------- reserve 与 resize ----------------
    void reserve(size_type new_cap) {   // 只改容量：保证 capacity() >= new_cap
        if (new_cap > _cap) {
            assert(new_cap <= max_size());
            _reallocate(new_cap);
        }
    }

    void resize(size_type new_size) {   // 改大小：变大默认构造，变小销毁尾部
        if (new_size < _size) {
            _destroy_range(_value + new_size, _value + _size);
            _size = new_size;
            return;
        }
        if (new_size == _size) return;
        _grow_if_needed(new_size);
        size_type i = _size;
        try { for (; i < new_size; ++i) _construct_at(_value + i); }
        catch (...) { _size = i; throw; }   // 已构造的部分计入 size，异常时析构不漏对象
        _size = new_size;
    }

    void resize(size_type new_size, const ValueT& value) {
        if (new_size < _size) {
            _destroy_range(_value + new_size, _value + _size);
            _size = new_size;
            return;
        }
        if (new_size == _size) return;
        _grow_if_needed(new_size);
        size_type i = _size;
        try { for (; i < new_size; ++i) _construct_at(_value + i, value); }
        catch (...) { _size = i; throw; }
        _size = new_size;
    }
```

`reserve(10)` 之后 `size()` 仍然是 0——它只换更大的块，绝不动已有元素，并且分配的是精确的 `new_cap`（调用者通常已算好需要多少）。`resize` 则真的改变元素个数：变大时逐个默认构造（带 `value` 的重载改为拷贝构造），变小时销毁尾部。

注意 `resize` 变大的异常处理：某个元素构造抛异常时，`catch` 里把 `_size` 设成 `i`（已构造成功的个数）再重新抛出，析构函数只会销毁真实存在的对象，既不漏也不多——这是**基本异常安全保证**的写法。另外，不带 `value` 的 `resize` 要求 `ValueT` 可默认构造，而 `push_back` 不要求，这是把「分配」和「构造」分开带来的好处。两个函数都不会缩小容量，想释放多余内存得另加 `shrink_to_fit`，本文从简。

### 元素访问

```cpp
    // ---------------- 元素访问 ----------------
    reference operator[](size_type i) noexcept { return _value[i]; }
    const_reference operator[](size_type i) const noexcept { return _value[i]; }

    reference at(size_type i) {
        if (i >= _size) throw std::out_of_range("ws_vector::at: index out of range");
        return _value[i];
    }
    const_reference at(size_type i) const {
        if (i >= _size) throw std::out_of_range("ws_vector::at: index out of range");
        return _value[i];
    }

    reference front() noexcept { assert(!empty()); return _value[0]; }
    const_reference front() const noexcept { assert(!empty()); return _value[0]; }
    reference back() noexcept { assert(!empty()); return _value[_size - 1]; }
    const_reference back() const noexcept { assert(!empty()); return _value[_size - 1]; }

    pointer data() noexcept { return _value; }
    const_pointer data() const noexcept { return _value; }
```

`operator[]` 不检查边界，和标准库一致，热路径上不背这个开销；要检查就用 `at()`，越界抛 `std::out_of_range`。下标类型是无符号的 `size_type`，`at(-1)` 会先转换成 `SIZE_MAX`，照样会被边界检查拦下。`front()`/`back()` 在空容器上是 UB，这里用 assert 兜底。`data()` 返回底层缓冲区首指针，也是下篇迭代器的桥梁。

一个容易忽略的细节：给 `at()` 加边界检查时，有人会写成 `if (index < 0 || index >= _size)`。对 `size_t` 来说 `index < 0` 恒假，是死代码，反而容易让人误以为负数下标被优雅处理了。

### Rule of Five：拷贝、移动与 noexcept

容器一旦管理资源，拷贝构造、拷贝赋值、移动构造、移动赋值、析构这五个特殊成员函数必须成套定义，缺一不可：

```cpp
    // ---------------- Rule of Five：拷贝 / 移动 ----------------
    ws_vector(const ws_vector& other) {
        _allocate_storage(other._size);
        try { for (; _size < other._size; ++_size) _construct_at(_value + _size, other._value[_size]); }
        catch (...) { _destroy_all_and_free(); throw; }
    }

    ws_vector& operator=(const ws_vector& other) {   // copy-and-swap，强异常安全
        if (this != &other) {
            ws_vector tmp(other);
            swap(tmp);
        }
        return *this;
    }

    ws_vector(ws_vector&& other) noexcept            // 直接偷走缓冲区，O(1)
        : _value(other._value), _size(other._size), _cap(other._cap) {
        other._value = nullptr;
        other._size = 0;
        other._cap = 0;
    }

    ws_vector& operator=(ws_vector&& other) noexcept {
        if (this != &other) {
            _destroy_range(_value, _value + _size);
            _deallocate();
            _value = other._value;
            _size = other._size;
            _cap = other._cap;
            other._value = nullptr;
            other._size = 0;
            other._cap = 0;
        }
        return *this;
    }
```

**拷贝构造**深拷贝：新分配装下 `other._size` 个元素的内存，逐个拷贝构造，失败清理逻辑与前面的构造函数相同。

**拷贝赋值**用 copy-and-swap：先拷出 `tmp`，成功了再交换。好处有两点——拷贝失败时 `tmp` 构造抛出，`*this` 根本没被碰过，天然强保证；自赋值也被 `this != &other` 挡住。如果按「先 `delete[]` 旧缓冲、再 `new` 新缓冲」的顺序写赋值，`new` 一旦抛异常，对象就烂掉了。copy-and-swap 唯一的代价是不能复用 `*this` 已有的容量，每次赋值都要新分配一次，对教学实现来说用一点性能换正确性是划算的。

**移动构造**直接接管三个成员再把 `other` 清空，O(1)；**移动赋值**先释放自己的资源（判自移动）再接管。

`noexcept` 不是装饰品，标准库真的会拿它做决策。`std::vector` 扩容时，如果元素移动构造是 `noexcept` 的，就放心用移动搬元素；否则为保住强异常安全，宁可退回拷贝——因为移动搬到一半抛异常，源容器已经被改烂，无法恢复。我们的 `_reallocate` 用 `std::move_if_noexcept` 表达同样的策略，测试第 6 节会验证。反过来，如果把会做内存分配的拷贝构造声明成 `noexcept`，分配失败时程序会直接 `std::terminate`——这是把编译期承诺用反了的典型错误。

### clear / erase / swap

```cpp
    // ---------------- clear / erase / swap ----------------
    void clear() noexcept {
        _destroy_range(_value, _value + _size);
        _size = 0;          // 只销毁元素，保留容量
    }

    void erase(size_type index) {
        assert(index < _size);   // 越界是 UB，和 std::vector::erase 的约定一致
        std::move(_value + index + 1, _value + _size, _value + index);
        --_size;
        (_value + _size)->~ValueT();
    }

    void swap(ws_vector& other) noexcept {
        std::swap(_value, other._value);
        std::swap(_size, other._size);
        std::swap(_cap, other._cap);
    }
```

`clear()` 只销毁元素、保留容量，下次 `push_back` 不用重新分配——与 `std::vector::clear()` 一致。如果想在 `clear` 里顺手释放缓冲区、把 `_cap` 置 0，务必同时把 `_value` 置空：漏掉这一步就留下悬垂指针，下次 `push_back` 走扩容路径时会对同一块内存重复 `delete[]`。

`erase(index)` 用 `std::move` 把后面的元素整体左移，再销毁最后一个位置，`_size` 减一，复杂度 O(n)，这是数组删除的固有代价。常见的错误写法是循环 `for (i = index; i < _size; ++i) _value[i] = _value[i + 1];`，它有两个独立问题：最后一次读 `_value[_size]` 越界，而且全程没有 `--_size`（删了等于没删）。

`swap` 只交换三个成员，O(1) 且 `noexcept`——这也是 copy-and-swap 能成立的前提，交换本身不允许失败。

### 分配 / 构造 / 销毁原语与数据成员

最后一块拼图是文件末尾的 private 区，它包含三个原语、扩容逻辑和三个成员：

```cpp
private:
    // ---------------- 分配 / 构造 / 销毁原语 ----------------
    static pointer _allocate(size_type n) {
        return static_cast<pointer>(::operator new(n * sizeof(ValueT)));
    }

    static void _destroy_range(pointer first, pointer last) noexcept {
        for (; first != last; ++first) first->~ValueT();
    }

    template <typename... Args>
    static void _construct_at(pointer p, Args&&... args) {
        ::new (static_cast<void*>(p)) ValueT(std::forward<Args>(args)...);
    }

    void _deallocate() noexcept {
        ::operator delete(_value);
        _value = nullptr;
        _cap = 0;
    }

    void _destroy_all_and_free() noexcept {
        _destroy_range(_value, _value + _size);
        _deallocate();
        _size = 0;
    }

    void _allocate_storage(size_type n) {   // 只在构造函数里调用：此时对象还是空的
        if (n == 0) return;
        _value = _allocate(n);
        _cap = n;
    }

    void _grow_if_needed(size_type needed) {
        if (needed <= _cap) return;
        assert(needed <= max_size());
        size_type new_cap = _cap == 0 ? 1 : _cap;
        while (new_cap < needed) new_cap *= 2;   // 每次翻倍是均摊 O(1) 的关键
        _reallocate(new_cap);
    }

    void _reallocate(size_type new_cap) {   // 全部搬完后才释放旧内存，强异常安全
        pointer new_value = _allocate(new_cap);
        size_type i = 0;
        try {
            for (; i < _size; ++i)
                ::new (static_cast<void*>(new_value + i)) ValueT(std::move_if_noexcept(_value[i]));
        } catch (...) {
            _destroy_range(new_value, new_value + i);
            ::operator delete(new_value);
            throw;
        }
        _destroy_range(_value, _value + _size);
        ::operator delete(_value);
        _value = new_value;
        _cap = new_cap;
    }

    // ---------------- 数据成员 ----------------
    pointer _value = nullptr;   // 缓冲区首地址
    size_type _size = 0;        // 已构造的元素个数
    size_type _cap = 0;         // 缓冲区容量
};
```

要点：

- `_allocate` 是纯分配，`_construct_at` 是纯构造，谁都不越界代劳；`_construct_at` 的参数包转发也是 `emplace_back` 省掉临时对象的底层原因。
- `_reallocate` 的顺序是「新块上搬完全部元素 → 才销毁旧元素、释放旧内存」。任何一步抛异常，旧缓冲区原封不动，容器保持强异常安全。
- `_grow_if_needed` 从 `_cap == 0` 时先给 1，之后每次翻倍直到满足 `needed`，容量轨迹是 1 → 2 → 4 → 8 → …，测试第 1 节能看到。
- `std::move_if_noexcept(x)` 的规则：`T` 的移动构造是 `noexcept`（或无法拷贝）时返回右值引用走移动，否则返回 `const T&` 走拷贝，保住强保证。

至此头文件完整结束。自由函数 `swap` 是给泛型代码准备的：

```cpp
template <typename ValueT>
void swap(ws_vector<ValueT>& a, ws_vector<ValueT>& b) noexcept {
    a.swap(b);
}

#endif  // WS_VECTOR_H
```

`std::swap(ws_vector, ws_vector)` 会走三次拷贝赋值的通用版本，而这个重载只换三个指针，O(1)，ADL 会优先找到它。

## 错误示范的 8 个典型问题与现场

下面这份错误示范（演示文件为 `bad_ws_vector.h`）在简单用例上能跑通，逐条检查却暗藏一串 bug：

| # | 错误示范的问题 | 后果 |
| --- | --- | --- |
| 1 | `erase` 循环写 `for (i = index; i < _size; ++i) _value[i] = _value[i + 1];`，且从不 `--_size` | 最后一次读 `_value[_size]` 越界；删完 size 不变，尾元素重复 |
| 2 | `resize` 的 `size >= _cap` 分支只调 `_expand_cap(size << 1)` | 不更新 `_size`、不构造新元素：`resize(10)` 后 size 仍是 0，容量倒是变成 20 |
| 3 | `clear()` `delete[] _value` 但不置空指针，还 `_cap = 0` | `_value` 悬垂；之后 `push_back` 走扩容路径，对同一块内存重复 `delete[]` |
| 4 | `pop_back()` 无条件 `--_size` | 空容器下溢成 `SIZE_MAX`，`size()` 变成 18446744073709551615 |
| 5 | `operator=` 先 `delete[]` 再 `new`，`new ValueT[_cap]` 要求默认构造，且只有拷贝赋值 | `new` 抛异常对象即烂；不能从临时对象赋值；`T` 必须默认构造 |
| 6 | 拷贝构造 `ws_vector(ws_vector& v) noexcept { *this = v; }` | `_value` 未初始化就被 `delete[]`（ASan: bad-free）；`noexcept` 却做分配 |
| 7 | `ws_vector(const ValueT&, size_t)` 没有调用 `init()`，`_value` 未初始化 | 第一次 `push_back` 就往野指针上写 |
| 8 | `swap` 借助拷贝赋值（三次深拷贝）；`at`/`resize` 里 `index < 0`、`size < 0` 对 `size_t` 恒假 | swap 是 O(n) 且可能抛异常；无效检查提供虚假安全感 |

口说无凭，用 ASan/UBSan 挑三种现场复现。第一种：容量刚好等于 `size` 时调用 `erase`。这种写法会触发如下报错：

```cpp
ws_vector<int> v;
for (int i = 1; i <= 4; ++i) v.push_back(i);   // capacity 恰为 4
v.erase(3);                                    // 循环里执行 _value[3] = _value[4]
```

```text
ERROR: AddressSanitizer: heap-buffer-overflow on address 0x602000000100 ...
READ of size 4 at 0x602000000100 thread T0
    #0 ws_vector<int>::erase(unsigned long) bad_ws_vector.h:71
0x602000000100 is located 0 bytes after 16-byte region [0x6020000000f0,0x602000000100)
SUMMARY: AddressSanitizer: heap-buffer-overflow bad_ws_vector.h:71 in ws_vector<int>::erase(unsigned long)
```

第二种：`clear()` 之后 `push_back`，触发 double free。这种写法会触发如下报错：

```text
ERROR: AddressSanitizer: attempting double-free on 0x6020000000d0 in thread T0:
    #0 _ZdaPv
    #1 ws_vector<int>::_expand_cap(unsigned long) bad_ws_vector.h:154
freed by thread T0 here:
    #1 ws_vector<int>::clear() bad_ws_vector.h:83
SUMMARY: AddressSanitizer: double-free bad_ws_vector.h:154 in ws_vector<int>::_expand_cap(unsigned long)
```

第三种不用 ASan 就能看到：`pop_back` 的 size 下溢，输出如下：

```text
pop 一次后 size=0
空容器再 pop 后 size=18446744073709551615
是否等于 SIZE_MAX: 1
```

这些问题的正确写法都体现在前面的实现里：`erase` 用 `std::move` + `--_size`；`clear` 保留容量并把资源管理收敛到 `_destroy_range`；`pop_back` 用 assert 拦截；`resize` 更新 `_size` 并处理构造异常；拷贝/移动按 Rule of Five 实现；`swap` 只交换成员。

## 编译与运行

测试覆盖六组场景：容量轨迹、`reserve`/`resize` 语义、增删改查、拷贝/移动语义、扩容失败的异常安全、`noexcept` 移动的调度。把 `ws_vector.h` 和下面的 `main.cpp` 放在同一目录：

```bash
clang++ -std=c++17 -Wall -Wextra -fsanitize=address,undefined -fno-omit-frame-pointer main.cpp -o main
./main
```

`main.cpp` 全文：

```cpp
#include "ws_vector.h"

#include <iostream>
#include <stdexcept>
#include <string>
#include <utility>

// 拷贝会抛异常的类型：验证扩容的强异常安全（没有移动构造，move_if_noexcept 退化为拷贝）
struct Bomb {
    static inline int alive = 0, countdown = 0;
    int value;
    explicit Bomb(int v) : value(v) { ++alive; }
    Bomb(const Bomb& o) : value(o.value) {
        if (--countdown < 0) throw std::runtime_error("Bomb: 拷贝时爆炸");
        ++alive;
    }
    ~Bomb() { --alive; }
};

// 移动构造 noexcept：验证扩容走移动而不是拷贝
struct MoveProbe {
    static inline int copies = 0, moves = 0;
    int value;
    explicit MoveProbe(int v) : value(v) {}
    MoveProbe(const MoveProbe& o) : value(o.value) { ++copies; }
    MoveProbe(MoveProbe&& o) noexcept : value(o.value) { ++moves; }
};

static void print_ints(const ws_vector<int>& v) {
    std::cout << '[';
    for (size_t i = 0; i < v.size(); ++i) std::cout << (i ? ", " : "") << v[i];
    std::cout << ']';
}

int main() {
    std::cout << "===== 1. push_back 与容量轨迹 =====\n";
    {
        ws_vector<int> v;
        std::cout << "初始: size=" << v.size() << " capacity=" << v.capacity() << "\n";
        for (int i = 0; i < 5; ++i) {
            v.push_back(i * i);
            std::cout << "push_back(" << i * i << ") -> size=" << v.size()
                      << " capacity=" << v.capacity() << "\n";
        }
    }

    std::cout << "\n===== 2. reserve 与 resize =====\n";
    {
        ws_vector<int> v;
        v.reserve(10);
        std::cout << "reserve(10):  size=" << v.size() << " capacity=" << v.capacity() << "\n";
        v.resize(3);
        std::cout << "resize(3):    size=" << v.size() << " capacity=" << v.capacity() << " -> ";
        print_ints(v);
        std::cout << "\n";
        v.resize(5, 42);
        std::cout << "resize(5,42): size=" << v.size() << " capacity=" << v.capacity() << " -> ";
        print_ints(v);
        std::cout << "\n";
        v.resize(2);
        std::cout << "resize(2):    size=" << v.size() << " capacity=" << v.capacity() << " -> ";
        print_ints(v);
        std::cout << "\n";
        try {
            v.at(100);
        } catch (const std::out_of_range& e) {
            std::cout << "at(100) 抛出异常: " << e.what() << "\n";
        }
        std::cout << "front=" << v.front() << " back=" << v.back()
                  << " data()[1]=" << v.data()[1] << "\n";
    }

    std::cout << "\n===== 3. erase / pop_back / clear =====\n";
    {
        ws_vector<int> v{1, 2, 3, 4, 5};
        std::cout << "初始:        "; print_ints(v); std::cout << "\n";
        v.pop_back();
        std::cout << "pop_back 后: "; print_ints(v); std::cout << "\n";
        v.erase(1);
        std::cout << "erase(1) 后: "; print_ints(v); std::cout << "\n";
        v.clear();
        std::cout << "clear 后: size=" << v.size() << " capacity=" << v.capacity() << "（容量保留）\n";
        v.push_back(99);
        std::cout << "再 push_back: "; print_ints(v); std::cout << "\n";
    }

    std::cout << "\n===== 4. 拷贝 / 移动语义 =====\n";
    {
        ws_vector<std::string> a{"alpha", "beta", "gamma"};
        ws_vector<std::string> b = a;   // 拷贝构造
        b[0] = "ALPHA";
        std::cout << "拷贝构造后 a[0]=" << a[0] << " b[0]=" << b[0] << "（互不影响）\n";

        ws_vector<std::string> c = std::move(a);   // 移动构造
        std::cout << "移动构造后 a.size=" << a.size() << " a.capacity=" << a.capacity()
                  << " c.size=" << c.size() << "\n";

        ws_vector<std::string> d;
        d = c;   // 拷贝赋值（copy-and-swap）
        d[1] = "BETA";
        std::cout << "拷贝赋值后 c[1]=" << c[1] << " d[1]=" << d[1] << "\n";

        ws_vector<std::string> e;
        e = std::move(d);   // 移动赋值
        std::cout << "移动赋值后 d.size=" << d.size() << " e.size=" << e.size() << "\n";

        ws_vector<std::string>& d_alias = d;
        d_alias = d;                     // 自赋值（用别名避免编译器的自赋值警告）
        ws_vector<std::string>& e_alias = e;
        e = std::move(e_alias);          // 自移动赋值
        std::cout << "自赋值后 e.size=" << e.size() << " e[0]=" << e[0] << "\n";
    }

    std::cout << "\n===== 5. 异常安全：扩容失败后容器原样保留 =====\n";
    {
        ws_vector<Bomb> v;
        v.reserve(2);
        v.emplace_back(1);
        v.emplace_back(2);
        std::cout << "扩容前: size=" << v.size() << " capacity=" << v.capacity() << "\n";

        Bomb::countdown = 1;   // 第 1 次拷贝成功，第 2 次抛异常
        try {
            v.push_back(Bomb(3));
        } catch (const std::runtime_error& e) {
            std::cout << "捕获异常: " << e.what() << "\n";
        }
        std::cout << "失败后: size=" << v.size() << " capacity=" << v.capacity()
                  << " 元素=" << v[0].value << "," << v[1].value << "\n";

        Bomb::countdown = 100;
        v.push_back(Bomb(3));
        std::cout << "恢复后: size=" << v.size() << " capacity=" << v.capacity()
                  << " 末尾=" << v.back().value << "\n";
    }
    std::cout << "Bomb 全部析构后 alive=" << Bomb::alive << "\n";

    std::cout << "\n===== 6. noexcept 移动让扩容走 move =====\n";
    {
        ws_vector<MoveProbe> v;
        v.reserve(4);
        for (int i = 0; i < 4; ++i) v.push_back(MoveProbe(i));
        MoveProbe::copies = 0;
        MoveProbe::moves = 0;
        v.push_back(MoveProbe(4));   // 容量 4 -> 8：搬 4 个旧元素 + 构造 1 个新元素
        std::cout << "push_back 触发扩容: copy=" << MoveProbe::copies
                  << " move=" << MoveProbe::moves << "\n";
    }

    return 0;
}
```

编译零警告，ASan/UBSan 全程无报错，输出如下：

```text
===== 1. push_back 与容量轨迹 =====
初始: size=0 capacity=0
push_back(0) -> size=1 capacity=1
push_back(1) -> size=2 capacity=2
push_back(4) -> size=3 capacity=4
push_back(9) -> size=4 capacity=4
push_back(16) -> size=5 capacity=8

===== 2. reserve 与 resize =====
reserve(10):  size=0 capacity=10
resize(3):    size=3 capacity=10 -> [0, 0, 0]
resize(5,42): size=5 capacity=10 -> [0, 0, 0, 42, 42]
resize(2):    size=2 capacity=10 -> [0, 0]
at(100) 抛出异常: ws_vector::at: index out of range
front=0 back=0 data()[1]=0

===== 3. erase / pop_back / clear =====
初始:        [1, 2, 3, 4, 5]
pop_back 后: [1, 2, 3, 4]
erase(1) 后: [1, 3, 4]
clear 后: size=0 capacity=5（容量保留）
再 push_back: [99]

===== 4. 拷贝 / 移动语义 =====
拷贝构造后 a[0]=alpha b[0]=ALPHA（互不影响）
移动构造后 a.size=0 a.capacity=0 c.size=3
拷贝赋值后 c[1]=beta d[1]=BETA
移动赋值后 d.size=0 e.size=3
自赋值后 e.size=3 e[0]=alpha

===== 5. 异常安全：扩容失败后容器原样保留 =====
扩容前: size=2 capacity=2
捕获异常: Bomb: 拷贝时爆炸
失败后: size=2 capacity=2 元素=1,2
恢复后: size=3 capacity=4 末尾=3
Bomb 全部析构后 alive=0

===== 6. noexcept 移动让扩容走 move =====
push_back 触发扩容: copy=0 move=5
```

三处输出值得单独看：

- 第 1 节：容量按 1 → 2 → 4 → 8 翻倍，`size` 只在构造成功后增长。
- 第 5 节：`Bomb::countdown = 1` 让第二次拷贝抛异常，扩容失败后 `size`、`capacity`、元素全部原样；恢复后继续 push 成功，作用域结束时 `alive=0`，没有一个对象泄漏——这正是 ASan 之外的第二重泄漏检查。
- 第 6 节：`MoveProbe` 的移动构造是 `noexcept`，扩容搬运 4 个旧元素加 1 个新元素，copy=0、move=5，`move_if_noexcept` 完全按预期调度。把它的移动构造改成可能抛异常（同时保留拷贝构造），这里会变成 copy=5——大对象上性能差别非常可观。

## 小结

- 动态数组的性能根基是连续内存和摊还 O(1) 的尾部插入；倍数扩容是摊还复杂度的关键，单次最坏 O(n) 无法避免。2 倍最省事，1.5 倍对分配器更友好。
- 把「分配内存」和「构造对象」分开是本实现的核心手法：`::operator new` + placement new 让元素不必默认构造，也避免了对未使用容量做无用构造。
- `resize` 改大小、`reserve` 只改容量；`clear` 保留容量；`pop_back`/`erase` 非法调用是 UB，debug 用 assert 兜底。
- Rule of Five 缺一不可：拷贝用 copy-and-swap 保强异常安全，移动标 `noexcept` 才能让泛型代码放心走移动，扩容用 `std::move_if_noexcept` 在「快」和「强保证」之间取舍。

现在的 `ws_vector` 只能靠下标访问，还是「数组」而不是「容器」。下篇《从零实现 C++ 动态数组（下）：迭代器与 STL 算法兼容》会给它补上符合 STL 规范的随机访问迭代器、`begin/end/cbegin/cend`，以及和 `std::sort`/`std::find`/`std::accumulate` 的配合，并讲清楚扩容与 `erase` 之后哪些迭代器会失效。
