---
title: C++ 内存拷贝：彻底搞懂 memcpy 与 memmove 的区别
date: 2021-07-23 19:18:30
updated: 2026-10-09
author: ws
description: 从零实现 memcpy/memmove，用图讲清内存重叠与拷贝方向
categories: ["C++"]
tags: ["C++", "内存"]
cover:
---

`memcpy` 和 `memmove` 的签名几乎一样，唯一区别是 `memmove` 允许源和目标内存区域重叠。真动手实现一遍就会发现：判断该往哪个方向拷、目标恰好等于源时怎么办、0 长度和空指针怎么处理，每个细节都有讲究。这篇文章从零实现这两个函数，用图解释重叠拷贝为什么会写坏数据，给出一份通过 ASan/UBSan 验证、和 libc 对拍的完整代码，最后说清楚标准库为什么通常比手写快，以及手写版本的教学价值在哪。

## 两个函数的契约只差一点：是否允许重叠

C 标准对 `memcpy` 的要求是：如果两块内存重叠，行为未定义（C11 §7.24.2.1）。`memmove` 则明确允许重叠，效果等价于"先把数据拷到临时数组，再拷到目标"（C11 §7.24.2.2），实现上当然不需要真的申请临时内存，只要选对拷贝方向。

| 函数 | 来源 | 是否允许重叠 | 常见用途 |
| --- | --- | --- | --- |
| `memcpy` | C11 §7.24.2.1 | 不允许，重叠即 UB | 两块独立缓冲区之间搬运 |
| `memmove` | C11 §7.24.2.2 | 允许 | 同一块缓冲区内前后挪动数据 |

既然 `memmove` 什么都能干，为什么还要保留 `memcpy`？一是契约更严格，调用方写错了更容易在测试阶段暴露；二是 C 标准里 `memcpy` 的参数带 `restrict` 限定，编译器可以放心用向量指令批量搬，`memmove` 没有这个优惠。实际上现代标准库的 `memmove` 也是先判断不重叠、再走和 `memcpy` 一样的快速路径，所以性能差距很小。

## 内存重叠到底有哪几种情形

把地址轴画出来，情况一目了然。下面的图里，向上是高地址方向：

```text
情形 A：完全不重叠，正向拷贝安全

    [src ───────── n ─────────]
                                  [dst ───── n ─────]


情形 B：dst < src，目标在低地址一侧，两段有交集

             [src ───────── n ─────────]
    [dst ───── n ─────]
    ↑ 正向拷贝时，写入位置永远落后于读取位置，不会碰到还没读的数据


情形 C：dst > src，目标在高地址一侧，两段有交集——危险

    [src ───────── n ─────────]
                   [dst ───── n ─────]
    ↑ 正向拷贝时，dst 的第一个字节就落在 src 区间内部，
      覆盖了后面还没读到的数据
```

结论：**只有情形 C（`src < dst < src + n`）会出事**。情形 B 虽然也重叠，但正向逐字节拷贝恰好是安全的；情形 A、`dst == src`、`dst == src + n`（刚好首尾相接）也都安全。

用一个具体的例子看情形 C 怎么把数据写坏。`buf = "abcde"`，现在想把前 4 个字符整体右移一格，即 `memcpy(buf + 1, buf, 4)`：

| 步骤 | 读取 | 写入 | 拷贝后的 `buf` |
| --- | --- | --- | --- |
| i=0 | `buf[0]='a'` | `buf[1]='a'` | `aacde` |
| i=1 | `buf[1]='a'` ← 已经被上一步覆盖 | `buf[2]='a'` | `aaade` |
| i=2 | `buf[2]='a'` ← 又被覆盖 | `buf[3]='a'` | `aaaae` |
| i=3 | `buf[3]='a'` ← 还是覆盖 | `buf[4]='a'` | `aaaaa` |

期望的结果是 `aabcd`，实际得到 `aaaaa`：第二次读取时读到的已经不再是 `'b'`，而是自己刚写进去的 `'a'`。`memmove` 遇到这种情况要从尾部开始倒着拷：先读 `buf[3]` 写 `buf[4]`，再读 `buf[2]` 写 `buf[3]`……每一步的读取位置都在写入位置之前，永远不会覆盖尚未读取的字节。

## 从零实现 ws_memcpy

先看最简单的一半。`memcpy` 按字节拷贝，用 `char*` 把 `void*` 拆开，从低地址到高地址逐字节走：

```cpp
// 要求 [src, src+n) 与 [dst, dst+n) 不重叠
inline void* ws_memcpy(void* dst, const void* src, size_t n)
{
    assert(dst != nullptr && src != nullptr);

    char* d = static_cast<char*>(dst);
    const char* s = static_cast<const char*>(src);
    while (n != 0) {
        *d++ = *s++;               // 从低地址向高地址逐字节拷贝
        --n;
    }
    return dst;
}
```

几点说明：

- 标准要求这两个函数返回 `dst`，方便链式调用，比如 `f(memcpy(a, b, n))`，所以最后 `return dst` 而不是返回 `d`。
- 用 `char*` 而不是 `T*`：`char` 是唯一没有对齐要求的类型之一，逐字节访问对任意来源的内存都合法，也不会踩到对象表示（object representation）的坑——直接对 `double` 数组按元素拷会在别名和严格别名规则上出问题。
- `while (n != 0)` 比 `while (n--)` 更好读：循环条件是"还有字节要拷"，自减放在循环体里意图更清晰；`n` 是无符号类型，`n--` 这种写法容易让人多想一秒。

## ws_memmove：先判方向，再决定从哪头拷

`memmove` 的全部难点就是方向判定。需要反向拷贝的条件是：目标落在源的开区间 `(src, src + n)` 里，也就是 `src < dst < src + n`。取反就是正向条件：`dst <= src` 或者 `dst >= src + n`。

不假思索的写法通常直接拿指针比大小：`d <= s || d >= s + n`。逻辑上没错，但这里藏着两个隐患：一是标准 C++ 只允许对同一数组内的指针做大小比较，两个独立对象之间的指针关系比较是未定义行为；二是 `s + n` 这一步本身可能加法溢出。更稳妥的做法是把指针转成 `uintptr_t` 再比较，并用差值 `dAddr - sAddr >= n` 代替 `sAddr + n`。由于 `||` 的短路特性，只有 `dAddr > sAddr` 时才会计算差值，也不会出现无符号下溢。

```cpp
inline void* ws_memmove(void* dst, const void* src, size_t n)
{
    assert(dst != nullptr && src != nullptr);

    char* d = static_cast<char*>(dst);
    const char* s = static_cast<const char*>(src);
    if (d == s || n == 0) {
        return dst;                // 自己拷自己、或空拷贝，直接返回
    }

    // 用整数比较地址：标准 C++ 不允许对指向不同对象的指针做大小比较
    const std::uintptr_t dAddr = reinterpret_cast<std::uintptr_t>(d);
    const std::uintptr_t sAddr = reinterpret_cast<std::uintptr_t>(s);

    if (dAddr <= sAddr || dAddr - sAddr >= n) {
        // 目标在源前面（低地址），或两段根本不相交：正向拷贝
        for (size_t i = 0; i < n; ++i) {
            d[i] = s[i];
        }
    } else {
        // 目标落在 (src, src+n) 内：从尾部开始反向拷贝
        for (size_t i = n; i != 0; --i) {
            d[i - 1] = s[i - 1];
        }
    }
    return dst;
}
```

边界逐条对一遍：

| 边界 | 走哪个分支 | 结果 |
| --- | --- | --- |
| `dst == src` | 开头直接返回 | 内容不变，正确；即使不做这个提前返回、按 `dAddr <= sAddr` 落到正向分支，逐字节自赋值同样正确 |
| `n == 0` | 开头直接返回 | 一个字节都不碰；注意开始处的 `assert` 仍然要求指针非空 |
| `dst == src + n` | 正向（`dAddr - sAddr == n`） | 两段刚好首尾相接，不重叠，正确 |
| `src < dst < src + n` | 反向 | 从尾部开始，读取永远领先于写入 |
| `dst < src` 且相交 | 正向 | 写入落在读取位置之前，安全 |

反向循环写成 `for (size_t i = n; i != 0; --i)` 而不是 `while (n--)`，也是为了避免无符号自减在 0 处的隐式回绕——虽然那个回绕恰好不执行循环体，但没必要依赖这种细节。

## 完整代码：ws_utilities.hpp

把两个函数、调试宏和随机数函数放进一个头文件。注意所有函数都加了 `inline`：头文件里定义的自由函数会被每个包含它的翻译单元各定义一份，少了 `inline`，两个以上的 `.cpp` 包含它时就会在链接期报重复定义。

```cpp
#ifndef WS_UTILITIES_HPP
#define WS_UTILITIES_HPP

// ---------------------------------------------------------------------------
// 内存泄漏检测：MSVC 调试堆是平台私有能力，用 _MSC_VER 隔离。
// _CRTDBG_MAP_ALLOC 必须在 <crtdbg.h> 之前定义，所以要放在所有 include 前面。
// ---------------------------------------------------------------------------
#ifdef _MSC_VER
#  define _CRTDBG_MAP_ALLOC          // 记录 malloc/new 的文件名与行号
#  include <crtdbg.h>
#  ifdef _DEBUG
#    define WS_DEBUG_NEW new (_NORMAL_BLOCK, __FILE__, __LINE__)
#  else
#    define WS_DEBUG_NEW new
#  endif
#else
#  define WS_DEBUG_NEW new
#endif

#include <cassert>
#include <cstddef>
#include <cstdint>
#include <random>
#include <utility>

// ---------------------------------------------------------------------------
// ws_memcpy：要求 [src, src+n) 与 [dst, dst+n) 不重叠
// ---------------------------------------------------------------------------
inline void* ws_memcpy(void* dst, const void* src, size_t n)
{
    assert(dst != nullptr && src != nullptr);

    char* d = static_cast<char*>(dst);
    const char* s = static_cast<const char*>(src);
    while (n != 0) {
        *d++ = *s++;               // 从低地址向高地址逐字节拷贝
        --n;
    }
    return dst;
}

// ---------------------------------------------------------------------------
// ws_memmove：允许两个区间重叠，自动选择拷贝方向
// ---------------------------------------------------------------------------
inline void* ws_memmove(void* dst, const void* src, size_t n)
{
    assert(dst != nullptr && src != nullptr);

    char* d = static_cast<char*>(dst);
    const char* s = static_cast<const char*>(src);
    if (d == s || n == 0) {
        return dst;                // 自己拷自己、或空拷贝，直接返回
    }

    // 用整数比较地址：标准 C++ 不允许对指向不同对象的指针做大小比较
    const std::uintptr_t dAddr = reinterpret_cast<std::uintptr_t>(d);
    const std::uintptr_t sAddr = reinterpret_cast<std::uintptr_t>(s);

    if (dAddr <= sAddr || dAddr - sAddr >= n) {
        // 目标在源前面（低地址），或两段根本不相交：正向拷贝
        for (size_t i = 0; i < n; ++i) {
            d[i] = s[i];
        }
    } else {
        // 目标落在 (src, src+n) 内：从尾部开始反向拷贝
        for (size_t i = n; i != 0; --i) {
            d[i - 1] = s[i - 1];
        }
    }
    return dst;
}

// ---------------------------------------------------------------------------
// ws_random：返回 [b, e] 闭区间内的均匀随机整数
// ---------------------------------------------------------------------------
inline int ws_random(int b, int e)
{
    if (b > e) {
        std::swap(b, e);
    }

    // 每个线程一份引擎，首次使用时初始化，避免每次调用都读熵
    static thread_local std::mt19937 engine = [] {
        std::random_device rd;
        std::seed_seq seq{rd(), rd(), rd(), rd(), rd(), rd(), rd(), rd()};
        return std::mt19937(seq);
    }();

    std::uniform_int_distribution<int> dist(b, e);   // 分布的区间每次可能不同
    return dist(engine);
}

#endif // WS_UTILITIES_HPP
```

## 验证：和 libc 对拍、边界用例、重叠演示

测试分四组：

1. **不重叠拷贝**：`ws_memcpy` 拷贝一个字符串，再和源字符串比对。
2. **穷举对拍**：在同一个 128 字节缓冲区里枚举源偏移、目标偏移和长度，`ws_memmove` 的结果必须和 `std::memmove` 逐字节一致。这一步覆盖了所有重叠组合，是整份测试里最有价值的部分。
3. **边界**：`n == 0` 必须返回 `dst` 且不碰内存，自拷贝必须不改变内容。
4. **随机数**：60 万次骰子，检查范围和均匀性。

```cpp
#include "ws_utilities.hpp"

#include <algorithm>
#include <cstdio>
#include <cstring>
#include <map>
#include <string>
#include <vector>

// ws_memcpy 处理不重叠区间
static bool test_memcpy_no_overlap()
{
    const char* src = "The quick brown fox jumps over the lazy dog";
    char dst[64] = {};
    ws_memcpy(dst, src, std::strlen(src) + 1);
    return std::strcmp(dst, src) == 0;
}

// ws_memmove 与 libc memmove 穷举对拍：
// 在同一个 128 字节 buffer 内枚举 src/dst 偏移与长度
static bool test_memmove_matches_libc()
{
    const size_t N = 128;
    std::vector<char> base(N), a(N), b(N);
    for (size_t i = 0; i < N; ++i) {
        base[i] = static_cast<char>(i);
    }

    for (size_t srcOff = 0; srcOff < N; ++srcOff) {
        for (size_t dstOff = 0; dstOff < N; ++dstOff) {
            const size_t room = N - std::max(srcOff, dstOff);
            const size_t maxLen = std::min(room, size_t(32));
            for (size_t len = 0; len <= maxLen; ++len) {
                a = base;
                b = base;
                std::memmove(a.data() + dstOff, a.data() + srcOff, len);
                ws_memmove(b.data() + dstOff, b.data() + srcOff, len);
                if (a != b) {
                    std::printf("  FAIL: srcOff=%zu dstOff=%zu len=%zu\n", srcOff, dstOff, len);
                    return false;
                }
            }
        }
    }
    std::printf("  (srcOff/dstOff 0..127 x len 0..32 全部一致)\n");
    return true;
}
```

后半部分是边界用例、重叠演示和随机数检查，最后 `main` 依次跑每项并打印结果：

```cpp
// 重叠场景演示：右移字符串
static void demo_overlap()
{
    char a[16] = "0123456789";
    ws_memcpy(a + 1, a, 9);                       // 故意用 memcpy 做重叠拷贝
    std::printf("  ws_memcpy 重叠右移 : %s\n", a); // 结果被污染

    char b[16] = "0123456789";
    ws_memmove(b + 1, b, 9);                      // memmove 处理重叠
    std::printf("  ws_memmove 重叠右移: %s\n", b);

    char c[16] = "0123456789";
    ws_memcpy(c, c + 1, 9);                       // 左移：正向拷贝恰好安全
    std::printf("  ws_memcpy 重叠左移 : %s\n", c);
}

// size == 0 与自拷贝
static bool test_edge_cases()
{
    char buf[8] = "abcdef";
    if (ws_memmove(buf, buf + 2, 0) != buf) return false;  // 空拷贝返回 dst
    if (ws_memcpy(buf, buf + 2, 0) != buf) return false;
    if (ws_memmove(buf, buf, 6) != buf) return false;      // 自拷贝不改变内容
    return std::strcmp(buf, "abcdef") == 0;
}

// ws_random 的范围与均匀性
static bool test_ws_random()
{
    std::map<int, int> hist;
    constexpr int kDraws = 600000;
    for (int i = 0; i < kDraws; ++i) {
        const int v = ws_random(1, 6);            // 模拟骰子
        if (v < 1 || v > 6) {
            std::printf("  FAIL: 越界 %d\n", v);
            return false;
        }
        ++hist[v];
    }
    const int expect = kDraws / 6;
    std::printf("  骰子 600000 次计数:");
    for (const auto& [face, count] : hist) {
        std::printf(" %d->%d", face, count);
        if (count < expect * 9 / 10 || count > expect * 11 / 10) {
            return false;                          // 偏离均匀分布 10% 以上视为失败
        }
    }
    std::printf("\n");
    std::printf("  十个 [0,100] 样本:");
    for (int i = 0; i < 10; ++i) {
        std::printf(" %d", ws_random(0, 100));
    }
    std::printf("\n");
    return true;
}

int main()
{
    struct Case {
        const char* name;
        bool (*fn)();
    };
    const Case cases[] = {
        {"ws_memcpy 不重叠拷贝", test_memcpy_no_overlap},
        {"ws_memmove 对拍 libc", test_memmove_matches_libc},
        {"边界：size==0 / 自拷贝", test_edge_cases},
        {"ws_random 范围与均匀性", test_ws_random},
    };

    int failed = 0;
    for (const Case& c : cases) {
        std::printf("[RUN ] %s\n", c.name);
        const bool ok = c.fn();
        std::printf("[%s] %s\n", ok ? "PASS" : "FAIL", c.name);
        failed += ok ? 0 : 1;
    }

    std::printf("\n重叠场景演示:\n");
    demo_overlap();

    std::printf("\n%s\n", failed == 0 ? "ALL TESTS PASSED" : "SOME TESTS FAILED");
    return failed == 0 ? 0 : 1;
}
```

编译时打开地址消毒器和未定义行为消毒器，任何越界读写、指针运算问题都会当场报出来：

```bash
clang++ -std=c++17 -Wall -Wextra -Wpedantic \
    -fsanitize=address,undefined -O1 -g \
    test_utilities.cpp -o test_utilities
./test_utilities
```

真实输出（Clang 21，Apple Silicon，macOS）：

```text
[RUN ] ws_memcpy 不重叠拷贝
[PASS] ws_memcpy 不重叠拷贝
[RUN ] ws_memmove 对拍 libc
  (srcOff/dstOff 0..127 x len 0..32 全部一致)
[PASS] ws_memmove 对拍 libc
[RUN ] 边界：size==0 / 自拷贝
[PASS] 边界：size==0 / 自拷贝
[RUN ] ws_random 范围与均匀性
  骰子 600000 次计数: 1->99951 2->100386 3->100340 4->99842 5->99700 6->99781
  十个 [0,100] 样本: 42 19 12 44 7 45 55 7 54 56
[PASS] ws_random 范围与均匀性

重叠场景演示:
  ws_memcpy 重叠右移 : 0000000000
  ws_memmove 重叠右移: 0012345678
  ws_memcpy 重叠左移 : 1234567899

ALL TESTS PASSED
```

输出里有三个值得看的点：

- 重叠右移时 `ws_memcpy` 把 `0123456789` 拷成了 `0000000000`，正是前面表格推导的"读到自己刚写的数据"；`ws_memmove` 则得到正确的 `0012345678`。
- 同为重叠，左移（`dst < src`）用 `ws_memcpy` 也能得到正确结果——这解释了为什么有些平台的 memcpy 恰好不崩，但**标准上它依旧是 UB，不能依赖**。
- 对拍用的是 `std::memmove`，穷举了几十万种偏移和长度组合，全部一致，说明方向判定没有漏掉边界。

## 为什么标准库通常比手写快

标准库的 `memmove`/`memcpy` 不只是"比你多写了几行优化"，它做的事情大致包括：

- **按机器字或 SIMD 宽度拷贝**：先处理头部几个不对齐的字节，主体一次搬 16/32/64 字节，再处理尾部。
- **给小尺寸专门优化**：几十字节以内直接用几组 load/store 拼出来，不建循环。
- **利用 `restrict` 语义**：`memcpy` 在 C 标准里声明的两个指针带 `restrict`，编译器知道它们不重叠，可以放心展开向量指令；手写循环里 `char*` 可能指向任何东西，编译器要么加运行期检查，要么放弃向量化。
- **针对具体 CPU 微架构调参**：非临时存储、缓存行对齐、prefetch 等。

那手写到底慢多少？在同一台机器上（Clang 21 `-O2`，Apple M 系列，源和目标都故意偏移 3 字节破坏对齐）实测：

```cpp
#include "ws_utilities.hpp"

#include <chrono>
#include <cstdio>
#include <cstring>
#include <vector>

// 对四种尺寸分别拷贝：先 std::memmove，再 ws_memmove
int main()
{
    struct Sample { size_t size; long iters; };
    const Sample samples[] = {
        {16, 20000000L}, {64, 10000000L}, {1024, 2000000L}, {1u << 20, 20000L},
    };

    for (const Sample& s : samples) {
        std::vector<char> src(s.size + 64, 0x5a);
        std::vector<char> dst(s.size + 64);
        char* d = dst.data() + 3;                 // 故意不按缓存行对齐
        const char* p = src.data() + 3;

        std::memmove(d, p, s.size);
        ws_memmove(d, p, s.size);

        const auto t0 = std::chrono::steady_clock::now();
        for (long i = 0; i < s.iters; ++i) std::memmove(d, p, s.size);
        const auto t1 = std::chrono::steady_clock::now();
        for (long i = 0; i < s.iters; ++i) ws_memmove(d, p, s.size);
        const auto t2 = std::chrono::steady_clock::now();

        const double msStd = std::chrono::duration<double, std::milli>(t1 - t0).count();
        const double msWs = std::chrono::duration<double, std::milli>(t2 - t1).count();
        const double total = double(s.size) * s.iters / (1024.0 * 1024.0 * 1024.0);
        std::printf("%7zu B | std::memmove %6.2f GiB/s | ws_memmove %6.2f GiB/s\n",
                    s.size, total / (msStd / 1000.0), total / (msWs / 1000.0));
        volatile char sink = dst[7];              // 防止整个循环被优化掉
        (void)sink;
    }
    return 0;
}
```

用 `clang++ -std=c++17 -O2 bench_memmove.cpp -o bench_memmove` 编译后，真实结果如下：

```text
     16 B | std::memmove   4.53 GiB/s | ws_memmove   8.95 GiB/s
     64 B | std::memmove  30.54 GiB/s | ws_memmove  45.68 GiB/s
   1024 B | std::memmove  60.70 GiB/s | ws_memmove  39.26 GiB/s
1048576 B | std::memmove  45.90 GiB/s | ws_memmove  45.86 GiB/s
```

结果和直觉不太一样，值得如实说明：小尺寸时手写版本反而快，因为函数被内联展开、循环被折叠；1 KiB 时 libc 快约 1.5 倍，靠的是 SIMD 批量拷贝；到 1 MiB 两者都撞上内存带宽，差距消失。这组数字说明：**现代编译器会把你手写的字节循环自动向量化，大块连续内存的拷贝差距没有想象中大；但标准库在各种尺寸、对齐、重叠组合下表现更稳定，还有 `restrict` 语义和平台调优背书**。生产代码直接用 `std::memcpy`/`std::memmove`（或 `__builtin_memcpy`），需要手写的场景主要是：没有标准库的裸机环境、需要处理特殊硬件缓冲区，以及——像本文这样，理解它到底是怎么工作的。

## 内存泄漏检测宏：MSVC 私有能力怎么跨平台

先看一种常见的错误写法：定义一个 `DEBUG_CHECK_MEMORY_LEAKS` 宏，配合 `<crtdbg.h>` 使用。它想做的是 MSVC 调试堆的内存跟踪，原理是：

- CRT 的调试堆会给每次分配附加一个头块，记录分配序号、文件名、行号；
- `new (_NORMAL_BLOCK, __FILE__, __LINE__) T` 这种带文件行号的 placement new 是 MSVC 特有的重载，把调用点的信息存进头块；
- 程序退出时如果设置了 `_CRTDBG_LEAK_CHECK_DF`，或显式调用 `_CrtDumpMemoryLeaks()`，CRT 会遍历未释放的块并打印来源文件行号。

这种写法有三个问题：`#include <crtdbg.h>` 无条件包含，非 Windows 平台直接编译失败；宏本身只展开成 `(_NORMAL_BLOCK, __FILE__, __LINE__)`，必须写成 `new DEBUG_CHECK_MEMORY_LEAKS T(...)` 才能用，和常见的 `DEBUG_NEW` 风格不同，忘记写 `new` 就是编译错误；此外它没有定义 `_CRTDBG_MAP_ALLOC`，`malloc` 一族的调用不会被记录文件行号。

推荐的跨平台写法里，非 MSVC 平台把它定义为普通 `new`，业务代码照常写 `Widget* w = WS_DEBUG_NEW Widget();`。MSVC 下还需要在 `main` 开头打开泄漏检查：

```cpp
#ifdef _MSC_VER
    _CrtSetDbgFlag(_CRTDBG_ALLOC_MEM_DF | _CRTDBG_LEAK_CHECK_DF);
#endif
```

这样程序退出时 CRT 会打印类似 `Detected memory leaks!` 的报告，标出分配地址、块大小和 `文件(行号)`；非 MSVC 平台这些宏展开为空，代码照常编译，只是没有泄漏检查。

跨平台的替代方案更省事，推荐优先用：

- **AddressSanitizer**：编译时加 `-fsanitize=address -g`，退出时自动打印泄漏报告和调用栈，Linux/macOS/Windows（Clang、GCC、MSVC 均支持）。本文的测试就是用 ASan 跑通的。
- **Valgrind**：`valgrind --leak-check=full ./prog`，不需要重新编译，但运行速度会慢一个数量级，且新 macOS 支持不完整，适合 Linux。
- **LeakSanitizer**：独立开启 `-fsanitize=leak`，只查泄漏，开销比全量 ASan 小。

## ws_random：随机数生成的几个常见错误

先看一种常见的错误写法：

```cpp
int ws_random(int b, int e)
{
    std::random_device sd;
    std::minstd_rand linearRan(sd());
    std::uniform_int_distribution<int> dis(b, e);
    return dis(linearRan);
}
```

能跑，但有几个问题：

1. **每次调用都构造 `std::random_device`**。它代表一个系统熵源，在部分平台上构造会打开设备或发起系统调用，代价远高于一次生成随机数；高频调用时性能会明显下降。
2. **`std::minstd_rand` 是最小的线性同余引擎**，随机质量和周期都一般，适合嵌入式省空间，不适合做通用随机。
3. 用单个 `sd()` 作为种子，种子的熵受限于随机引擎的种子宽度。

推荐的实现把引擎放到 `thread_local` 里，只在每个线程首次调用时初始化，并用 `seed_seq` 把 8 次 `random_device` 的输出铺开当种子：

```cpp
inline int ws_random(int b, int e)
{
    if (b > e) {
        std::swap(b, e);
    }

    static thread_local std::mt19937 engine = [] {
        std::random_device rd;
        std::seed_seq seq{rd(), rd(), rd(), rd(), rd(), rd(), rd(), rd()};
        return std::mt19937(seq);
    }();

    std::uniform_int_distribution<int> dist(b, e);   // 分布的区间每次可能不同
    return dist(engine);
}
```

要点：`std::uniform_int_distribution` 每次调用重新构造，因为区间可能不同；`thread_local` 保证多线程下各用各的引擎，无需加锁；`std::mt19937` 周期 2^19937-1，做游戏、模拟、测试数据都绰绰有余。测试输出里 60 万次骰子的计数偏差在 1% 以内，符合均匀分布的预期。

## 踩坑与边界清单

1. **`memcpy` 重叠是 UB，不是"可能拷错"**。x86 上碰巧左移正确、右移数据错乱，都只是某个实现的副作用；编译器一旦把它优化成 SIMD 拷贝，越界写入、数据全错、甚至崩溃都可能发生。移动同一块缓冲区里的数据，永远用 `memmove`。
2. **空指针断言是调试契约，不是运行时保护**。`assert` 在 `NDEBUG` 下会被编译掉，release 版本传空指针照样崩。另外按标准，即使 `n == 0`，传入空指针也是未定义行为，所以本文的 `assert` 不区分 `n` 是否为 0。如果确实需要运行时防护，得写显式的 `if` 分支并定义好错误处理方式（抛异常、返回错误码），但那就偏离标准库语义了。
3. **头文件里的自由函数必须 `inline`**。忘了写 `inline`，单个 `.cpp` 能通过，两个以上包含它的翻译单元就会在链接期报 "duplicate symbol"。
4. **指针大小比较要用 `uintptr_t`**。`p1 < p2` 在指向不同对象时是未定义行为，`reinterpret_cast<uintptr_t>` 之后再比较是标准做法；差值比较还能顺手避免 `src + n` 的溢出。
5. **拷贝的是字节数，不是元素个数**。`ws_memcpy(arr, src, 10)` 只拷 10 字节，不是 10 个 `int`；自身不管理类型，这一点和 `std::copy` 完全不同。
6. **别把 `memcpy` 用在重叠的 POD 结构体上**，也别用 `memcmp` 比较含填充字节的结构体——填充字节的值是不确定的。这类需求应该交给 `std::bit_cast` 或显式的字段比较。

## 总结

- `memcpy` 不处理重叠，`memmove` 处理：方向判定的核心是"目标是否落在 `(src, src + n)` 里"，是则从尾部反向拷贝。
- 反向拷贝的条件可以用 `dst <= src || dst >= src + n` 表达；用 `uintptr_t` 做比较、用差值代替加法判断，能同时避开指针比较 UB 和整数溢出。
- `dst == src`、`dst == src + n`、`n == 0` 三个边界都不需要特殊分支，但显式处理能让意图更清楚；空指针断言只在调试构建有效。
- 标准库快在字长/SIMD、对齐处理、`restrict` 语义和平台调优；现代编译器也能把手写循环向量化，大块内存拷贝差距有限，但库函数在边界组合上更可靠。
- 泄漏检查优先用 ASan 或 Valgrind；MSVC 的 `crtdbg` 方案要用 `#ifdef _MSC_VER` 隔离，并配上 `_CRTDBG_MAP_ALLOC` 与 `_CrtSetDbgFlag` 才能真正生效。
- 随机数不要每次调用都构造 `random_device`；`thread_local` 引擎 + `seed_seq` 是既安全又高效的写法。
