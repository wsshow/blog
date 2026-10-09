---
title: 从零实现 C++ 动态数组（下）：迭代器与 STL 算法兼容
date: 2021-08-06 22:18:30
updated: 2026-10-09
author: ws
description: 给自研 vector 实现符合 STL 规范的随机访问迭代器，让它能配合 sort/find/range-for
categories: ["C++"]
tags: ["C++", "迭代器", "数据结构"]
cover:
---

上篇《从零实现 C++ 动态数组（上）：内存管理与扩容策略》把 `ws_vector` 做到了「能用下标访问」的程度：分配与构造分离、2 倍扩容、Rule of Five、强异常安全。但它的接口只有 `v[0]`、`v.size()` 这类写法：不能 `for (int x : v)`，也不能直接交给 `std::sort`。这篇给它补上符合 STL 规范的随机访问迭代器——**本文默认你已读过上篇**，理解 `_value/_size/_cap`、扩容和移动语义，代码接着上篇的 `ws_vector.h` 写，只展示新增部分，完整合并版放在文末。

完成后这些代码都能跑：

```cpp
ws_vector<int> v{5, 2, 8, 1};
std::sort(v.begin(), v.end());       // 排序
auto it = std::find(v.begin(), v.end(), 8);   // 查找
for (int x : v) std::cout << x;      // range-for
```

## 迭代器是什么

一句话：**迭代器是指针的泛化**。指针能做的事（解引用、自增、比较、相减），迭代器都在类里重新实现一遍，但不再要求底层是连续数组——链表、树、哈希表都可以提供自己的迭代器。算法只依赖这套统一的操作，不需要知道容器的内部结构，这就是「容器与算法解耦」：`std::sort` 只有一份实现，`std::vector`、`std::deque` 都能用；哪天你写了一个新容器，只要迭代器满足要求，标准库的全部算法立刻可用。

STL 迭代器区间采用**左闭右开**约定 `[begin, end)`：`begin` 指向第一个元素，`end` 指向最后一个元素的「下一个位置」，对这个位置解引用是非法的。

```text
begin()                                    end()
  │                                          │
  ▼                                          ▼
  +------+------+------+------+------+
  | 10   | 20   | 30   | 40   | 50   |
  +------+------+------+------+------+
   [begin, end) 覆盖全部 5 个元素，end() 在最后一个元素的后面一格
```

这样约定有两个好处：空容器可以统一表示为 `[begin, begin)`；遍历循环统一写成 `for (; first != last; ++first)`——不需要单独处理「最后一个元素」这个边界。`end()` 只是地址合法的「哨兵」，任何算法都不会解引用它。

## iterator_traits 的五个类型

算法拿到一个迭代器 `It`，需要知道关于它的五件事。C++ 的约定是把这些类型定义成迭代器类的嵌套 `typedef`，再用 `std::iterator_traits<It>` 统一取出来（对裸指针 `T*`，标准库有专门特化，相当于 `T*` 也满足这套契约）：

| 类型 | 含义 | 算法为什么需要 |
| --- | --- | --- |
| `value_type` | 元素类型（不含 const） | 声明临时变量，如 `std::sort` 里的 pivot |
| `difference_type` | 有符号的差值类型 | `std::distance` 的返回类型、`it + n` 里 n 的类型 |
| `pointer` | 指针类型 `T*` | 实现 `operator->` |
| `reference` | 解引用返回类型 `T&` / `const T&` | 实现 `operator*`、`operator[]` |
| `iterator_category` | 能力标签（输入/前向/双向/随机访问） | 算法按标签选择实现，比如 `std::sort` 要求随机访问 |

`iterator_category` 是最容易被写错的一个：它不是在描述「你能做什么」，而是算法用来做**重载分发的承诺**。声明了 `random_access_iterator_tag`，算法就会用 `last - first`、`it + n` 这些假设去操作你；反过来，明明实现了随机访问却声明成双向，`std::sort` 会直接拒绝编译（它内部需要做 `last - first`）。

## 目标：LegacyRandomAccessIterator 的完整操作清单

一个符合 LegacyRandomAccessIterator 的迭代器需要支持：

- 可拷贝构造、可拷贝赋值、可析构、**可默认构造**；
- `*r`（解引用）、`r->m`（成员访问）、`++r`、`r++`、`--r`、`r--`；
- `r += n`、`r -= n`、`r + n`、`n + r`、`r - n`；
- `b - a`，返回 `difference_type`；
- `a[n]`，等价于 `*(a + n)`；
- 六种比较：`==`、`!=`、`<`、`<=`、`>`、`>=`；
- `iterator_traits` 五个类型全部正确。

先看一段不假思索写出的迭代器实现——表面上能用，逐条对照契约却踩了一串坑：

```cpp
typedef std::bidirectional_iterator_tag iterator_category;  // ① 标签写低了

reference operator*() { return *_v; }                       // ② 非 const 成员
pointer operator->() const { return &(operator*()); }       // ③ const 成员调用非 const 的 *

_Self operator+(size_t n) {                                 // ④ 循环自增，O(n)
    while (n--) ++_v;
    return *this;
}

size_t operator-(const iterator& o) {                       // ⑤ 返回无符号数
    return _v - o._v;
}
```

| 问题 | 后果 |
| --- | --- |
| ① 支持 `+`/`-` 却声明 `bidirectional_iterator_tag` | `std::sort`、`std::lower_bound` 拒绝编译 |
| ② 只有一套迭代器，`operator*` 还不是 const 成员 | 没有 `const_iterator`；`for (int x : const容器)`、const 迭代器解引用全部失败 |
| ③ `operator->` 是 const 却调用非 const 的 `operator*` | const 迭代器上 `it->x` 编译错误；即使能编译，const 正确性也是错的 |
| ④ `operator+`/`operator-`（减 n）用 `while (n--)` 循环 | 每次移动 O(n)；参数是无符号 `size_t`，`it + (-1)` 会变成天文数字 |
| ⑤ 差值返回 `size_t` | 与 traits 里的 `difference_type`（`ptrdiff_t`）不一致；`it1 - it2` 为负时包装成巨大正数 |
| 缺 `+=`、`-=`、`[]`、`<`、`<=`、`>`、`>=` | 不满足随机访问迭代器契约，`it < end` 之类的常见写法编译不过 |
| `iterator() {}` 不初始化 `_v` | 默认构造出的迭代器内容是垃圾，拷贝后比较是 UB |
| 只有 `begin()`/`end()` 非 const 版本 | 不能在 const 容器上用；range-for 也要求 const 版本 |

要补的坑比这段代码本身还多，与其逐个打补丁，不如一开始就按完整契约实现。下面的推荐实现直接覆盖整张清单。

## 实现 iterator 与 const_iterator

有两种写法：写两个独立的类（`iterator` 和 `const_iterator`），或者用**一个模板参数化指针类型**。后者代码少一半、两者之间的转换天然免费，代价是编译报错时类型名更长（`iterator_impl<const int*>` 之类）。标准库实现（libstdc++ 的 `normal_iterator`、libc++ 的指针包装）都选单模板，本文也用它：

```cpp
    // ---------------- 迭代器（下篇新增） ----------------
    // Ptr = ValueT* 时是 iterator，Ptr = const ValueT* 时是 const_iterator
    template <typename Ptr>
    class iterator_impl {
        template <typename> friend class iterator_impl;
    public:
        using iterator_category = std::random_access_iterator_tag;
        using value_type        = std::remove_const_t<std::remove_pointer_t<Ptr>>;
        using difference_type   = std::ptrdiff_t;
        using pointer           = Ptr;
        using reference         = decltype(*std::declval<Ptr>());

        iterator_impl() noexcept : _ptr(nullptr) {}
        explicit iterator_impl(Ptr ptr) noexcept : _ptr(ptr) {}

        // 非 const 迭代器可以隐式转成 const 迭代器，反过来不行
        template <typename OtherPtr, typename = std::enable_if_t<std::is_convertible<OtherPtr, Ptr>::value>>
        iterator_impl(const iterator_impl<OtherPtr>& other) noexcept : _ptr(other._ptr) {}

        reference operator*() const { return *_ptr; }
        pointer operator->() const { return _ptr; }
        reference operator[](difference_type n) const { return _ptr[n]; }

        iterator_impl& operator++() { ++_ptr; return *this; }
        iterator_impl operator++(int) { iterator_impl tmp = *this; ++_ptr; return tmp; }
        iterator_impl& operator--() { --_ptr; return *this; }
        iterator_impl operator--(int) { iterator_impl tmp = *this; --_ptr; return tmp; }

        // 随机访问：全部是 O(1) 的指针运算
        iterator_impl& operator+=(difference_type n) { _ptr += n; return *this; }
        iterator_impl& operator-=(difference_type n) { _ptr -= n; return *this; }
        iterator_impl operator+(difference_type n) const { return iterator_impl(_ptr + n); }
        iterator_impl operator-(difference_type n) const { return iterator_impl(_ptr - n); }
        friend iterator_impl operator+(difference_type n, iterator_impl it) { return it + n; }

        template <typename OtherPtr>
        difference_type operator-(const iterator_impl<OtherPtr>& o) const { return _ptr - o._ptr; }

        template <typename OtherPtr> bool operator==(const iterator_impl<OtherPtr>& o) const { return _ptr == o._ptr; }
        template <typename OtherPtr> bool operator!=(const iterator_impl<OtherPtr>& o) const { return _ptr != o._ptr; }
        template <typename OtherPtr> bool operator<(const iterator_impl<OtherPtr>& o) const { return _ptr < o._ptr; }
        template <typename OtherPtr> bool operator>(const iterator_impl<OtherPtr>& o) const { return _ptr > o._ptr; }
        template <typename OtherPtr> bool operator<=(const iterator_impl<OtherPtr>& o) const { return _ptr <= o._ptr; }
        template <typename OtherPtr> bool operator>=(const iterator_impl<OtherPtr>& o) const { return _ptr >= o._ptr; }
    private:
        Ptr _ptr;
    };

    using iterator = iterator_impl<ValueT*>;
    using const_iterator = iterator_impl<const ValueT*>;
```

几个设计点：

- `value_type` 用 `std::remove_const_t<std::remove_pointer_t<Ptr>>` 从指针类型反推元素类型：`const int*` 推出的也是 `int`（traits 规定 `value_type` 不带 const）。
- `reference` 用 `decltype(*std::declval<Ptr>())`：`Ptr = int*` 时是 `int&`，`Ptr = const int*` 时是 `const int&`。`operator*`、`operator->` 都标了 const，const 正确性从类型层面自动成立（错误示范里的 ②③ 两个问题在这里一并解决）。
- **转换构造函数**用 `std::enable_if_t<std::is_convertible<OtherPtr, Ptr>::value>` 约束：`int*` 能转 `const int*`，所以 `iterator` 隐式转 `const_iterator`；反方向不可转换，SFINAE 直接屏蔽，不会出现「const 迭代器偷偷转成可写迭代器」的危险。
- **比较和求差写成成员模板**（`template <typename OtherPtr>`），于是 `iterator` 和 `const_iterator` 可以任意方向混合比较、混合求差，不需要写两套重载。
- **`n + it` 写成类内的 friend 函数**而不是类外模板：类外写 `template <typename Ptr> iterator_impl<Ptr> operator+(typename iterator_impl<Ptr>::difference_type, ...)`，第一个参数处于「非推导上下文」，`1 + it` 推导不出 `Ptr` 而编译失败；friend 定义在类内、靠 ADL 找到，没有这个问题。
- 所有随机访问运算都是 `_ptr + n`、`_ptr - n` 这样的指针算术，O(1)；参数是 `difference_type`（有符号），`it - 1` 合法——错误示范里 ④⑤ 两个问题在这里都不存在。

## begin / end / cbegin / cend 与 range-for

容器侧只需要暴露六个函数：

```cpp
    iterator begin() noexcept { return iterator(_value); }
    iterator end() noexcept { return iterator(_value + _size); }
    const_iterator begin() const noexcept { return const_iterator(_value); }
    const_iterator end() const noexcept { return const_iterator(_value + _size); }
    const_iterator cbegin() const noexcept { return const_iterator(_value); }
    const_iterator cend() const noexcept { return const_iterator(_value + _size); }
```

非 const 容器上调 `begin()` 得到可写的 `iterator`；const 容器上只能调用 const 重载，得到 `const_iterator`。`cbegin()/cend()` 则不管容器是不是 const，总是返回 `const_iterator`——C++11 引入它们，是为了让「我只想读」的意图在非 const 容器上也能明确表达。

有了这六个函数，range-for 的 `for (int x : v)` 就能用了。编译器把 range-for 展开成大致这样的代码：

```cpp
{
    auto __begin = v.begin();
    auto __end   = v.end();
    for (; __begin != __end; ++__begin) {
        int x = *__begin;   // 声明类型是 int，就从 *__begin 拷贝初始化一个 int
        // 循环体
    }
}
```

也就是说，range-for 对我们的迭代器只要求四件事：`begin`/`end` 能被找到、`!=` 能比较、`++` 能前进、`*` 能解引用。这四条恰恰是迭代器最基础的契约，我们全部满足。

## 迭代器版 erase：返回下一个位置

上篇的 `erase(size_t index)` 按索引删，返回 `void`。标准库的 `erase` 接收迭代器并返回**被删区间之后那个新位置的迭代器**——这个返回值在「边遍历边删」时非常有用：

```cpp
    // 删除 [first, last)：尾部元素整体左移，返回被删区间原来的位置
    iterator erase(const_iterator first, const_iterator last) {
        const difference_type n = last - first;
        assert(n >= 0);
        pointer dst = _value + (first - cbegin());
        pointer src = _value + (last - cbegin());
        std::move(src, _value + _size, dst);
        _destroy_range(_value + _size - static_cast<size_type>(n), _value + _size);
        _size -= static_cast<size_type>(n);
        return iterator(dst);
    }

    iterator erase(const_iterator pos) { return erase(pos, pos + 1); }

    // 上篇的按下标删除接口，内部委托给迭代器版本
    iterator erase(size_type index) { return erase(cbegin() + static_cast<difference_type>(index)); }
```

实现和上篇同源：`std::move` 把 `[last, end)` 的元素整体左移到 `first`，销毁尾部多余对象，`_size` 减去删除个数，最后返回指向新位置 `dst` 的迭代器。参数类型用 `const_iterator`，普通 `iterator` 可以隐式转过来——这也是标准库 C++11 之后的选择。遍历中删除的惯用法因此变成：

```cpp
for (auto it = v.begin(); it != v.end(); ) {
    if (要删) it = v.erase(it);   // erase 返回下一个有效位置
    else      ++it;
}
```

## 用 STL 算法验证

验证分两层：编译期用 `static_assert` 检查 `iterator_traits` 的五个类型，运行期用 `std::sort`、`std::find`、`std::accumulate`、`std::distance`、`std::lower_bound`、`std::reverse` 这些对迭代器要求各不相同的算法，再用 range-for 和 `const_iterator` 各走一遍。测试前半部分：

```cpp
#include "ws_vector.h"

#include <algorithm>
#include <cctype>
#include <cstddef>
#include <cstdint>
#include <iostream>
#include <iterator>
#include <numeric>
#include <string>
#include <type_traits>

using IntVec = ws_vector<int>;

// 编译期检查：iterator_traits 的五个类型必须齐全且类型正确
static_assert(std::is_same_v<std::iterator_traits<IntVec::iterator>::value_type, int>);
static_assert(std::is_same_v<std::iterator_traits<IntVec::iterator>::difference_type, std::ptrdiff_t>);
static_assert(std::is_same_v<std::iterator_traits<IntVec::iterator>::pointer, int*>);
static_assert(std::is_same_v<std::iterator_traits<IntVec::iterator>::reference, int&>);
static_assert(std::is_same_v<std::iterator_traits<IntVec::iterator>::iterator_category,
                             std::random_access_iterator_tag>);
static_assert(std::is_same_v<std::iterator_traits<IntVec::const_iterator>::reference, const int&>);
static_assert(!std::is_same_v<IntVec::iterator, IntVec::const_iterator>);
```

`iterator_traits` 对自定义迭代器会去取类里的嵌套 typedef；如果某一条缺失或者类型不对，这些 `static_assert` 会在编译期直接失败。测试主体：

```cpp
int main() {
    std::cout << "===== 1. STL 算法：sort / find / accumulate / lower_bound / reverse =====\n";
    {
        ws_vector<int> v{5, 2, 8, 1, 9, 3};
        std::sort(v.begin(), v.end());
        std::cout << "sort 后: ";
        for (int x : v) std::cout << x << ' ';
        std::cout << "\n";

        auto it = std::find(v.begin(), v.end(), 8);
        std::cout << "find(8): 下标=" << (it - v.begin()) << "\n";

        std::cout << "accumulate=" << std::accumulate(v.begin(), v.end(), 0)
                  << " distance=" << std::distance(v.begin(), v.end()) << "\n";

        auto lb = std::lower_bound(v.begin(), v.end(), 5);
        std::cout << "lower_bound(5): 下标=" << (lb - v.begin()) << " 值=" << *lb << "\n";

        std::reverse(v.begin(), v.end());
        std::cout << "reverse 后: ";
        for (int x : v) std::cout << x << ' ';
        std::cout << "\n";
    }

    std::cout << "\n===== 2. 随机访问迭代器算术 =====\n";
    {
        ws_vector<int> v{10, 20, 30, 40, 50};
        auto it = v.begin() + 2;
        std::cout << "begin()+2 = " << *it << "\n";
        it += 1;
        std::cout << "+=1       = " << *it << "\n";
        it -= 2;
        std::cout << "-=2       = " << *it << "\n";
        std::cout << "it[3]     = " << it[3] << "\n";
        std::cout << "end()-begin() = " << (v.end() - v.begin()) << "\n";
        std::cout << "2+begin()     = " << *(2 + v.begin()) << "\n";
        std::cout << std::boolalpha;
        std::cout << "it < end()    = " << (it < v.end()) << "\n";
        std::cout << "it >= begin() = " << (it >= v.begin()) << "\n";

        IntVec::const_iterator cit = v.begin() + 1;   // iterator -> const_iterator
        std::cout << "const_iterator 解引用 = " << *cit
                  << "，与 iterator 比较 = " << (cit == v.begin() + 1) << "\n";
    }

    std::cout << "\n===== 3. range-for 与 const 容器 =====\n";
    {
        ws_vector<std::string> words{"pear", "apple", "orange"};
        std::sort(words.begin(), words.end());

        std::cout << "排序后: ";
        for (const std::string& w : words) std::cout << w << ' ';
        std::cout << "\n";

        for (std::string& w : words)
            w[0] = static_cast<char>(std::toupper(static_cast<unsigned char>(w[0])));
        std::cout << "首字母大写: ";
        for (const std::string& w : words) std::cout << w << ' ';
        std::cout << "\n";

        const ws_vector<std::string>& cw = words;
        std::cout << "const 容器: ";
        for (auto it = cw.cbegin(); it != cw.cend(); ++it) std::cout << *it << ' ';
        std::cout << "\n";
    }

    std::cout << "\n===== 4. erase 的返回值 =====\n";
    {
        ws_vector<int> v{10, 20, 30, 40, 50};
        auto it = std::find(v.begin(), v.end(), 20);
        it = v.erase(it);   // 返回指向被删元素之后那个位置的迭代器
        std::cout << "erase(20) 后 *it=" << *it << " size=" << v.size() << "\n";

        auto first = std::find(v.begin(), v.end(), 30);
        v.erase(first, first + 2);   // 删掉 30、40
        std::cout << "区间 erase 后: ";
        for (int x : v) std::cout << x << ' ';
        std::cout << "\n";
    }

    std::cout << "\n===== 5. 迭代器失效演示（只比较地址，不解引用失效迭代器）=====\n";
    {
        ws_vector<int> v;
        v.reserve(2);
        v.push_back(1);
        v.push_back(2);
        const std::uintptr_t before = reinterpret_cast<std::uintptr_t>(v.data());

        v.push_back(3);   // 容量 2 -> 4，缓冲区搬家
        const bool moved = reinterpret_cast<std::uintptr_t>(v.data()) != before;
        std::cout << std::boolalpha << "push_back 触发扩容后底层缓冲区更换: " << moved << "\n";

        auto it = v.begin();
        std::cout << "重新取 begin() 解引用: " << *it << "\n";

        auto pos = std::find(v.begin(), v.end(), 3);
        pos = v.erase(pos);
        std::cout << "erase(3) 返回的迭代器: "
                  << (pos == v.end() ? std::string("end()") : std::to_string(*pos)) << "\n";
        std::cout << "erase 后 distance=" << std::distance(v.begin(), v.end()) << "\n";
    }

    return 0;
}
```

编译和上篇同款命令：

```bash
clang++ -std=c++17 -Wall -Wextra -fsanitize=address,undefined -fno-omit-frame-pointer main.cpp -o main
./main
```

零警告，ASan/UBSan 无报错，输出：

```text
===== 1. STL 算法：sort / find / accumulate / lower_bound / reverse =====
sort 后: 1 2 3 5 8 9 
find(8): 下标=4
accumulate=28 distance=6
lower_bound(5): 下标=3 值=5
reverse 后: 9 8 5 3 2 1 

===== 2. 随机访问迭代器算术 =====
begin()+2 = 30
+=1       = 40
-=2       = 20
it[3]     = 50
end()-begin() = 5
2+begin()     = 30
it < end()    = true
it >= begin() = true
const_iterator 解引用 = 20，与 iterator 比较 = true

===== 3. range-for 与 const 容器 =====
排序后: apple orange pear 
首字母大写: Apple Orange Pear 
const 容器: Apple Orange Pear 

===== 4. erase 的返回值 =====
erase(20) 后 *it=30 size=4
区间 erase 后: 10 50 

===== 5. 迭代器失效演示（只比较地址，不解引用失效迭代器）=====
push_back 触发扩容后底层缓冲区更换: true
重新取 begin() 解引用: 1
erase(3) 返回的迭代器: end()
erase 后 distance=2
```

每一节都在验证不同的契约：`std::sort` 要求随机访问 + 可交换；`std::find` 只要求输入迭代器；`std::accumulate` 和 `std::distance` 依赖 traits 里的 `value_type`/`difference_type`；第 2 节把 `+`、`+=`、`-=`、`[]`、`n + it`、`<`、`>=` 和迭代器到 `const_iterator` 的隐式转换全部走了一遍；`erase` 的返回值在第 4 节确认指向 30、区间删除后剩 `[10, 50]`。

## 迭代器失效规则

迭代器本质是「指向缓冲区某个位置」的薄包装（我们的实现里就是一个裸指针），所以缓冲区一变，迭代器就可能变成悬垂指针。规则与 `std::vector` 完全一致：

| 操作 | 迭代器失效范围 | 引用 / 指针失效范围 |
| --- | --- | --- |
| `push_back` / `emplace_back` 触发扩容 | **全部失效**（包括 `end()`） | 全部失效 |
| `push_back` 未触发扩容 | 仅 `end()` 失效 | 不失效 |
| `reserve` 且容量真的变大 | **全部失效** | 全部失效 |
| `resize` 变小 | 指向被销毁元素的迭代器失效 | 同左 |
| `erase(pos)` / `erase(first, last)` | `pos`（或 `first`）及其后的所有迭代器失效，`end()` 也会变 | 同左 |
| `clear` | 全部失效 | 全部失效 |
| `swap` | 迭代器仍然有效，但指向「对方容器里的元素」 | 同左 |

因为扩容是「在新缓冲区构造、释放旧缓冲区」，所以只要 `data()` 的地址变了，旧指针、旧引用、旧迭代器全部作废。测试第 5 节只做地址比较、不解引用失效迭代器，就是刻意为避免 UB：

```cpp
const std::uintptr_t before = reinterpret_cast<std::uintptr_t>(v.data());
v.push_back(3);   // 容量 2 -> 4，缓冲区搬家
const bool moved = reinterpret_cast<std::uintptr_t>(v.data()) != before;   // true
// 此时旧的 it / data() / v[0] 的引用都不能再用了，必须重新取
```

对比 `std::vector`：规则一模一样，标准只额外保证 `swap` 之后迭代器「跟随元素」到另一个容器，以及 `reserve(不小于当前容量)` 不失效。写循环时记住两条惯用法即可：**任何可能扩容的操作之后重新取迭代器；`erase` 用返回值续接遍历**。

## 附录：完整合并头文件

正文只展示了迭代器相关的新增部分。把上篇的 `ws_vector.h` 与这些内容合并后，完整头文件如下——多出来的部分就是迭代器、`begin/end/cbegin/cend` 和迭代器版 `erase`，配合上面的 `main.cpp` 即可编译：

```cpp
#ifndef WS_VECTOR_H
#define WS_VECTOR_H

// 从零实现 std::vector（下篇）：完整合并版头文件 ws_vector.h（C++17）
#include <algorithm>
#include <cassert>
#include <cstddef>
#include <initializer_list>
#include <iterator>
#include <limits>
#include <stdexcept>
#include <type_traits>
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

    // ---------------- 迭代器（下篇新增） ----------------
    // Ptr = ValueT* 时是 iterator，Ptr = const ValueT* 时是 const_iterator
    template <typename Ptr>
    class iterator_impl {
        template <typename> friend class iterator_impl;
    public:
        using iterator_category = std::random_access_iterator_tag;
        using value_type        = std::remove_const_t<std::remove_pointer_t<Ptr>>;
        using difference_type   = std::ptrdiff_t;
        using pointer           = Ptr;
        using reference         = decltype(*std::declval<Ptr>());

        iterator_impl() noexcept : _ptr(nullptr) {}
        explicit iterator_impl(Ptr ptr) noexcept : _ptr(ptr) {}

        // 非 const 迭代器可以隐式转成 const 迭代器，反过来不行
        template <typename OtherPtr, typename = std::enable_if_t<std::is_convertible<OtherPtr, Ptr>::value>>
        iterator_impl(const iterator_impl<OtherPtr>& other) noexcept : _ptr(other._ptr) {}

        reference operator*() const { return *_ptr; }
        pointer operator->() const { return _ptr; }
        reference operator[](difference_type n) const { return _ptr[n]; }

        iterator_impl& operator++() { ++_ptr; return *this; }
        iterator_impl operator++(int) { iterator_impl tmp = *this; ++_ptr; return tmp; }
        iterator_impl& operator--() { --_ptr; return *this; }
        iterator_impl operator--(int) { iterator_impl tmp = *this; --_ptr; return tmp; }

        // 随机访问：全部是 O(1) 的指针运算
        iterator_impl& operator+=(difference_type n) { _ptr += n; return *this; }
        iterator_impl& operator-=(difference_type n) { _ptr -= n; return *this; }
        iterator_impl operator+(difference_type n) const { return iterator_impl(_ptr + n); }
        iterator_impl operator-(difference_type n) const { return iterator_impl(_ptr - n); }
        friend iterator_impl operator+(difference_type n, iterator_impl it) { return it + n; }

        template <typename OtherPtr>
        difference_type operator-(const iterator_impl<OtherPtr>& o) const { return _ptr - o._ptr; }

        template <typename OtherPtr> bool operator==(const iterator_impl<OtherPtr>& o) const { return _ptr == o._ptr; }
        template <typename OtherPtr> bool operator!=(const iterator_impl<OtherPtr>& o) const { return _ptr != o._ptr; }
        template <typename OtherPtr> bool operator<(const iterator_impl<OtherPtr>& o) const { return _ptr < o._ptr; }
        template <typename OtherPtr> bool operator>(const iterator_impl<OtherPtr>& o) const { return _ptr > o._ptr; }
        template <typename OtherPtr> bool operator<=(const iterator_impl<OtherPtr>& o) const { return _ptr <= o._ptr; }
        template <typename OtherPtr> bool operator>=(const iterator_impl<OtherPtr>& o) const { return _ptr >= o._ptr; }
    private:
        Ptr _ptr;
    };

    using iterator = iterator_impl<ValueT*>;
    using const_iterator = iterator_impl<const ValueT*>;

    iterator begin() noexcept { return iterator(_value); }
    iterator end() noexcept { return iterator(_value + _size); }
    const_iterator begin() const noexcept { return const_iterator(_value); }
    const_iterator end() const noexcept { return const_iterator(_value + _size); }
    const_iterator cbegin() const noexcept { return const_iterator(_value); }
    const_iterator cend() const noexcept { return const_iterator(_value + _size); }

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

    // ---------------- 容量 ----------------
    bool empty() const noexcept { return _size == 0; }
    size_type size() const noexcept { return _size; }
    size_type capacity() const noexcept { return _cap; }
    size_type max_size() const noexcept { return std::numeric_limits<size_type>::max() / sizeof(ValueT); }

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
        assert(!empty());
        --_size;
        (_value + _size)->~ValueT();
    }

    // ---------------- reserve 与 resize ----------------
    void reserve(size_type new_cap) {
        if (new_cap > _cap) {
            assert(new_cap <= max_size());
            _reallocate(new_cap);
        }
    }

    void resize(size_type new_size) {
        if (new_size < _size) {
            _destroy_range(_value + new_size, _value + _size);
            _size = new_size;
            return;
        }
        if (new_size == _size) return;
        _grow_if_needed(new_size);
        size_type i = _size;
        try { for (; i < new_size; ++i) _construct_at(_value + i); }
        catch (...) { _size = i; throw; }
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

    // ---------------- Rule of Five：拷贝 / 移动 ----------------
    ws_vector(const ws_vector& other) {
        _allocate_storage(other._size);
        try { for (; _size < other._size; ++_size) _construct_at(_value + _size, other._value[_size]); }
        catch (...) { _destroy_all_and_free(); throw; }
    }

    ws_vector& operator=(const ws_vector& other) {
        if (this != &other) {
            ws_vector tmp(other);
            swap(tmp);
        }
        return *this;
    }

    ws_vector(ws_vector&& other) noexcept
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

    // ---------------- clear / erase / swap ----------------
    void clear() noexcept {
        _destroy_range(_value, _value + _size);
        _size = 0;
    }

    // 删除 [first, last)：尾部元素整体左移，返回被删区间原来的位置
    iterator erase(const_iterator first, const_iterator last) {
        const difference_type n = last - first;
        assert(n >= 0);
        pointer dst = _value + (first - cbegin());
        pointer src = _value + (last - cbegin());
        std::move(src, _value + _size, dst);
        _destroy_range(_value + _size - static_cast<size_type>(n), _value + _size);
        _size -= static_cast<size_type>(n);
        return iterator(dst);
    }

    iterator erase(const_iterator pos) { return erase(pos, pos + 1); }

    // 上篇的按下标删除接口，内部委托给迭代器版本
    iterator erase(size_type index) { return erase(cbegin() + static_cast<difference_type>(index)); }

    void swap(ws_vector& other) noexcept {
        std::swap(_value, other._value);
        std::swap(_size, other._size);
        std::swap(_cap, other._cap);
    }

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

    void _allocate_storage(size_type n) {
        if (n == 0) return;
        _value = _allocate(n);
        _cap = n;
    }

    void _grow_if_needed(size_type needed) {
        if (needed <= _cap) return;
        assert(needed <= max_size());
        size_type new_cap = _cap == 0 ? 1 : _cap;
        while (new_cap < needed) new_cap *= 2;
        _reallocate(new_cap);
    }

    void _reallocate(size_type new_cap) {
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

    pointer _value = nullptr;
    size_type _size = 0;
    size_type _cap = 0;
};

template <typename ValueT>
void swap(ws_vector<ValueT>& a, ws_vector<ValueT>& b) noexcept {
    a.swap(b);
}

#endif  // WS_VECTOR_H
```

## 小结

- 迭代器是指针的泛化，左闭右开 `[begin, end)`；它把容器和算法解耦，代价是要严格遵守 traits 五要素和随机访问操作清单。
- 错误示范里那套迭代器的核心问题：能力标签写成双向却实现了随机访问、`operator+/-` 是 O(n) 循环、差值返回无符号数、缺 `const_iterator`/`+=`/`-=`/`[]`/完整比较、`operator->` 的 const 正确性错误、默认构造不初始化指针——这些都属于「不满足 STL 契约」，算法层面立刻暴露。
- 用单个 `iterator_impl<Ptr>` 模板同时实现 `iterator` 和 `const_iterator`：`enable_if` 约束的单向隐式转换 + 成员模板比较，代码量减半，混合比较两个方向都可用。
- 验证方式是编译期 `static_assert` + 运行期 `std::sort`/`std::find`/`std::accumulate`/`std::distance`，再叠加 ASan/UBSan。
- 失效规则和 `std::vector` 一致：凡扩容必全失效；`erase` 之后用返回的迭代器续接遍历，不要继续使用已失效的迭代器。

至此 `ws_vector` 系列完成：上篇解决了内存与扩容，本篇解决了接口与算法兼容。想继续深入，可以试着给 `ws_vector` 加上 `insert`、`emplace`、`shrink_to_fit`，或者把 `_reallocate` 的增长策略换成 1.5 倍再跑一遍扩容轨迹。
