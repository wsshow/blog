---
title: 用 GoogleTest 给自研容器写单元测试
date: 2021-08-06 23:18:30
updated: 2026-10-09
author: ws
description: 从 CMake 集成到夹具与参数化测试，给 ws_vector/ws_list 建立测试套件
categories: ["测试"]
tags: ["C++", "GoogleTest", "单元测试"]
cover:
---

自研容器是单元测试最该盯住的代码：内存申请释放、扩容搬移、迭代器失效、拷贝控制，随便哪个环节出错都可能在特定输入下才爆。这篇文章从零搭一个跨平台的 CMake + GoogleTest 工程，逐条设计 ws_vector/ws::list 的测试用例，讲清 TEST/TEST_F/TEST_P 和死亡测试的用法，并用 ASan 和覆盖率数据验证测试真的走到了关键路径。测试针对系列前两篇实现的容器，先按那两篇的说明把 `ws_vector.h` 和 `ws_list.hpp` 拼好放进工程根目录。

## 为什么自研容器必须测

自研容器和普通业务代码不一样：它直接管理裸内存，错误往往不会在当下报出来。几类典型问题：

- **生命周期错误**：`new`/`delete` 不配对、placement new 之后忘记显式析构、扩容搬移了元素却没有析构旧位置。小规模数据看不出问题，`std::string` 这类非平凡类型一进来就崩。
- **扩容边界**：容量翻倍瞬间指针失效、搬移过程中丢元素、`size`/`capacity` 不同步。
- **空容器**：`front()`、`back()`、`pop_back()` 在空容器上的行为是重灾区，断言、异常还是未定义行为，必须明确。
- **拷贝控制**：自赋值、移动后的对象还能不能用、是深拷贝还是浅拷贝。
- **越界**：`operator[]` 不检查（和 `std::vector` 一致），`at()` 必须抛 `std::out_of_range`。

先看一段常见的错误示范，问题不少：只对拍了 `size()`，元素错位、丢数据都发现不了；用随机数生成数据，失败时连输入都不知道；`v_std[pos]` 里的 `pos` 取 `0..100`，而容器里只有 100 个元素，`pos == 100` 就越界访问——测试自己踩了 UB；`#pragma comment(lib, ...)` 是 MSVC 私有写法，换平台直接失效；用例写成 `test_all()` 却没有任何入口调用它。下面逐个拆解正确的写法。

## 用 CMake + FetchContent 引入 GoogleTest

先看一种常见的错误写法：用预处理宏拼出要链接的库文件路径。它除了把工程绑死在 MSVC 上，还有一个直接编译不过的问题：

```cpp
#define GTEST_LIB_PATH(path) "../GTest/"##path
#pragma comment(lib, GTEST_LIB_PATH("gtest.lib"))
```

`##` 是 token 粘贴运算符，把字符串字面量（`"../GTest/"`）和宏参数粘在一起会产生无效的预处理 token，Clang 直接报 `pasting formed ... an invalid preprocessing token`。字符串拼接本来就该写成 `"../GTest/" path`（相邻字面量自动连接），或者交给 CMake 管理。跨平台做法是用 `FetchContent` 在配置阶段自动克隆并编译 GoogleTest：

```cmake
cmake_minimum_required(VERSION 3.16)
project(ws_test LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 用 FetchContent 在配置阶段自动拉取 GoogleTest，
# 不再需要手写 #pragma comment(lib, ...)
include(FetchContent)
FetchContent_Declare(
  googletest
  GIT_REPOSITORY https://github.com/google/googletest.git
  GIT_TAG        v1.18.0
  GIT_SHALLOW    TRUE
)
# MSVC 下让 gtest 和我们的测试共用同一份 CRT，避免运行库冲突
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

enable_testing()
include(GoogleTest)

add_executable(ws_test
  test_ws_vector.cpp
  test_ws_list.cpp
)

target_compile_options(ws_test PRIVATE -Wall -Wextra)
target_link_libraries(ws_test PRIVATE GTest::gtest_main)

# 构建时枚举用例，注册给 ctest；每加一个 TEST 都会自动出现在 ctest 里
gtest_discover_tests(ws_test)
```

几个关键点：

- `GIT_TAG v1.18.0` 锁定版本，避免上游更新导致构建行为变化；`GIT_SHALLOW` 只拉最新一次提交，克隆更快。
- 链接 `GTest::gtest_main` 就自带 `main()`：它内部调用 `testing::InitGoogleTest()` 和 `RUN_ALL_TESTS()`，不需要手写入口。如果自己写一个 `test_all()` 却又忘了调用，测试就会一次都不跑。
- `gtest_discover_tests` 在构建完成后运行一次测试程序，把每个 `TEST` 注册成独立的 ctest 用例；交叉编译场景可以加 `DISCOVERY_MODE PRE_TEST` 改成运行时发现。
- 配置时加 `-DCMAKE_BUILD_TYPE=Debug`：默认构建类型不带优化也不定义 `NDEBUG`，断言生效，死亡测试才能按预期工作。

工程目录结构：

```text
ws_test/
├── CMakeLists.txt
├── ws_vector.h       ← 前篇《从零实现 C++ 动态数组（上）》
├── ws_list.hpp       ← 前篇《从零实现 C++ 双向链表》
├── test_ws_vector.cpp
└── test_ws_list.cpp
```

测试只依赖两容器以下接口：`ws_vector<T>` 的 `push_back` / `pop_back` / `erase(index)` / `operator[]` / `at` / `front` / `back` / `data` / `size` / `capacity` / `empty` 和拷贝移动；`ws::list<T>` 的 `push_back` / `push_front` / `pop_front` / `pop_back` / `front` / `back` / `contains` / `size` / `empty` / 双向迭代器和拷贝移动。

配置和构建：

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build -j 8
```

真实输出节选（AppleClang 21，macOS）：`-- Found Threads: TRUE`，然后 `-- Configuring done` / `-- Generating done`，构建产物在 `build/ws_test`。

## TEST、断言与夹具

**TEST** 定义一个用例：第一个参数是测试套件名，第二个是用例名，输出里显示为 `套件名.用例名`。**TEST_F** 使用夹具：派生 `::testing::Test`，成员数据会被 GTest 为每个用例重建一份，用例之间互不影响；`SetUp()`/`TearDown()` 在用例前后执行。

断言分两档，行为差异很关键：

| 系列 | 失败后的行为 | 使用场景 |
| --- | --- | --- |
| `EXPECT_*` | 记录失败，当前用例继续执行 | 大多数检查；一次跑完能看到所有错误 |
| `ASSERT_*` | 记录失败，立即从当前函数返回 | 后续代码依赖该检查时，比如断言非空再解引用 |

`ASSERT_*` 只从**当前函数**返回；封装在辅助函数里时，失败只会结束辅助函数，调用方还会继续。用两个故意失败的用例实测：

```cpp
#include <gtest/gtest.h>

TEST(FailDemo, ExpectKeepsRunning)
{
    int x = 1;
    EXPECT_EQ(x, 2);   // 失败后用例继续执行
    EXPECT_EQ(x, 3);   // 这一条也会执行并报告
}

TEST(FailDemo, AssertStops)
{
    int x = 1;
    ASSERT_EQ(x, 2);   // 失败后直接结束本用例
    EXPECT_EQ(x, 3);   // 不会执行
}
```

运行时 `ExpectKeepsRunning` 报告源文件第 6、7 行两条 Failure，`AssertStops` 只报告第 13 行一条就结束——这正是"失败是否中止当前用例"的区别。

容器测试的骨架：`WsVectorSmoke` 演示无夹具的 `TEST`；`WsVectorTest` 是夹具，每个用例都从一份全新的 `{1, 2, 3, 4, 5}` 开始。

```cpp
#include "ws_vector.h"   // 来自《从零实现 C++ 动态数组（上）》

#include <algorithm>
#include <string>
#include <vector>

#include <gtest/gtest.h>

// 确定性伪随机序列：每次运行输入相同，失败可复现。
// 直接用随机数的话，失败时连输入都不知道是什么。
static int sampleValue(int i) { return (i * 37 + 11) % 101 - 50; }

// TEST：无夹具的独立用例
TEST(WsVectorSmoke, PushBackKeepsOrder)
{
    ws_vector<int> v;
    for (int i = 0; i < 5; ++i) v.push_back(i * i);
    ASSERT_EQ(v.size(), 5u);
    for (int i = 0; i < 5; ++i) EXPECT_EQ(v[i], i * i);   // EXPECT 失败后继续跑
}

// TEST_F：每个用例都从夹具里的 {1,2,3,4,5} 开始
class WsVectorTest : public ::testing::Test {
protected:
    ws_vector<int> v{1, 2, 3, 4, 5};
};
```

## 用例设计：把容器的契约钉死在测试里

**空容器**：`empty()`、`size()`、`data()` 必须一致；放入一个元素后 `front()`/`back()` 都指向它；弹出后回到空。这里刻意不调用空容器的 `front()`，那是违反前置条件的用法，交给死亡测试验证。

```cpp
TEST_F(WsVectorTest, FrontBackAndPop)
{
    EXPECT_EQ(v.front(), 1);
    EXPECT_EQ(v.back(), 5);

    v.pop_back();
    EXPECT_EQ(v.size(), 4u);
    EXPECT_EQ(v.back(), 4);
}

TEST_F(WsVectorTest, EmptyContainerBasics)
{
    ws_vector<int> empty;
    EXPECT_TRUE(empty.empty());
    EXPECT_EQ(empty.size(), 0u);
    EXPECT_EQ(empty.data(), nullptr);             // 空容器不持有缓冲区

    empty.push_back(7);
    EXPECT_FALSE(empty.empty());
    EXPECT_EQ(empty.front(), 7);
    EXPECT_EQ(empty.back(), 7);

    empty.pop_back();
    EXPECT_TRUE(empty.empty());
}
```

**扩容与容量**：连续推入 1000 个确定性数据，每一步都和 `std::vector` 对拍。只比较 `size()` 不够——错位、丢元素、内容损坏都要逐元素比较才能发现。容器目前没有迭代器（迭代器是系列下篇的内容），用 `data()` 和大小构造区间配合 `std::equal`。

```cpp
TEST_F(WsVectorTest, GrowthPreservesValuesAndCapacity)
{
    std::vector<int> ref{1, 2, 3, 4, 5};
    for (int i = 0; i < 1000; ++i) {
        const int value = sampleValue(i);
        ref.push_back(value);
        v.push_back(value);
    }

    ASSERT_EQ(v.size(), ref.size());
    EXPECT_GE(v.capacity(), v.size());
    EXPECT_LT(v.capacity(), 2 * v.size());        // 翻倍扩容：capacity < 2 * size
    EXPECT_TRUE(std::equal(v.data(), v.data() + v.size(), ref.begin()));
}
```

**erase**：删中间元素后 `size` 减一，后面的元素整体前移。`erase` 接收下标、返回 void，验证方式就是比较删除后的完整内容。

```cpp
TEST_F(WsVectorTest, EraseShiftsAndShrinks)
{
    v.erase(1);                                   // 删掉元素 2
    EXPECT_EQ(v.size(), 4u);

    const int expected[] = {1, 3, 4, 5};
    EXPECT_TRUE(std::equal(v.data(), v.data() + v.size(), std::begin(expected)));
}
```

**越界**：`at()` 必须抛 `std::out_of_range`。用随机下标调用 `operator[]` 是常见的错误示范：越界时不报错反而继续跑，测试毫无意义。

```cpp
TEST_F(WsVectorTest, AtThrowsOnOutOfRange)
{
    EXPECT_EQ(v.at(0), 1);
    EXPECT_EQ(v.at(4), 5);
    EXPECT_THROW(v.at(5), std::out_of_range);     // 越界必须抛，不能读脏内存
    EXPECT_THROW(v.at(100), std::out_of_range);
}
```

**自赋值与普通赋值**：`v = v` 走的是提前返回的短路分支，不能替代对拷贝/移动赋值主体逻辑的测试，两类都要写。

```cpp
TEST_F(WsVectorTest, SelfAssignmentKeepsObjectValid)
{
    v = v;
    EXPECT_EQ(v.size(), 5u);
    EXPECT_EQ(v.back(), 5);

    v = std::move(v);                             // 自移动不能把自己清空
    EXPECT_EQ(v.size(), 5u);
    EXPECT_EQ(v.front(), 1);
}

TEST_F(WsVectorTest, AssignmentCopiesAndMoves)
{
    ws_vector<int> copy;
    copy = v;                                     // 普通拷贝赋值
    ASSERT_EQ(copy.size(), 5u);
    copy[0] = 100;
    EXPECT_EQ(v[0], 1);                           // 深拷贝，互不影响

    ws_vector<int> moved;
    moved.push_back(9);                           // 目标里先放点东西
    moved = std::move(copy);                      // 普通移动赋值，覆盖原有内容
    EXPECT_EQ(moved.size(), 5u);
    EXPECT_EQ(moved[0], 100);
    EXPECT_TRUE(copy.empty());
}
```

**拷贝/移动后源对象可用性**：对副本的改动不能影响源对象；被移动对象必须处于可析构、可继续使用的状态。注意 `std::vector` 被移动后的状态是"有效但未指定"，这里是针对我们自己容器的明确约定。

```cpp
TEST_F(WsVectorTest, CopyAndMoveLeaveSourceUsable)
{
    ws_vector<int> copy(v);
    ASSERT_EQ(copy.size(), v.size());
    copy.push_back(6);
    EXPECT_EQ(v.size(), 5u);                      // 拷贝与原对象互不影响

    ws_vector<int> moved(std::move(copy));
    EXPECT_EQ(moved.size(), 6u);
    EXPECT_EQ(moved.back(), 6);
    EXPECT_TRUE(copy.empty());                    // 移动后源对象为空
    copy.push_back(42);                           // 被移动对象可以继续使用
    EXPECT_EQ(copy.size(), 1u);
    EXPECT_EQ(copy[0], 42);
}
```

**非平凡类型**：`int` 测不出析构和深拷贝的 bug，换成 `std::string` 再来一轮。`erase` 忘记析构、拷贝只复制指针，ASan 会直接报泄漏或重复释放。

```cpp
TEST(WsVectorStringTest, ManagesNonTrivialElements)
{
    ws_vector<std::string> v;
    v.push_back("hello");
    v.push_back("world");
    v.erase(0);
    ASSERT_EQ(v.size(), 1u);
    EXPECT_EQ(v[0], "world");

    ws_vector<std::string> copy = v;              // 深拷贝
    copy.push_back("!");
    EXPECT_EQ(v.size(), 1u);
    EXPECT_EQ(copy.size(), 2u);
}
```

`ws::list` 的重点是两端操作、迭代器和拷贝控制，`contains` 则是 `ws_vector` 没有的接口：

```cpp
#include "ws_list.hpp"   // 来自《从零实现 C++ 双向链表》

#include <algorithm>
#include <list>

#include <gtest/gtest.h>

TEST(WsListTest, PushBackMatchesStdList)
{
    std::list<int> ref;
    ws::list<int> ws;

    for (int i = 0; i < 100; ++i) {
        const int value = (i * 37 + 11) % 101;    // 确定性数据，失败可复现
        ref.push_back(value);
        ws.push_back(value);
    }

    ASSERT_EQ(ws.size(), ref.size());
    EXPECT_TRUE(std::equal(ws.begin(), ws.end(), ref.begin()));
}

TEST(WsListTest, PushFrontReversesOrder)
{
    ws::list<int> ws;
    for (int i = 0; i < 5; ++i) ws.push_front(i); // 依次插入 0..4
    ASSERT_EQ(ws.size(), 5u);
    EXPECT_EQ(ws.front(), 4);
    EXPECT_EQ(ws.back(), 0);

    const int expected[] = {4, 3, 2, 1, 0};
    EXPECT_TRUE(std::equal(ws.begin(), ws.end(), std::begin(expected)));
}

TEST(WsListTest, PopFromBothEnds)
{
    ws::list<int> ws;
    for (int i = 1; i <= 4; ++i) ws.push_back(i); // 1 2 3 4

    ws.pop_front();                               // 2 3 4
    ws.pop_back();                                // 2 3
    ASSERT_EQ(ws.size(), 2u);
    EXPECT_EQ(ws.front(), 2);
    EXPECT_EQ(ws.back(), 3);

    ws.pop_front();
    ws.pop_back();
    EXPECT_TRUE(ws.empty());
    EXPECT_EQ(ws.size(), 0u);
}

TEST(WsListTest, CopyAndMoveAreIndependent)
{
    ws::list<int> ws;
    ws.push_back(1);
    ws.push_back(2);

    ws::list<int> copy(ws);
    copy.push_back(3);
    EXPECT_EQ(ws.size(), 2u);                     // 深拷贝
    EXPECT_TRUE(copy.contains(3));

    ws::list<int> moved(std::move(copy));
    EXPECT_EQ(moved.size(), 3u);
    EXPECT_TRUE(copy.empty());                    // 被移动后回到空状态
    copy.push_back(9);                            // 仍可继续使用
    EXPECT_EQ(copy.size(), 1u);
}

TEST(WsListTest, AssignmentAndMoveAssignment)
{
    ws::list<int> ws;
    ws.push_back(1);
    ws.push_back(2);

    ws::list<int> other;
    other = ws;                                   // 拷贝赋值
    other.push_back(3);
    EXPECT_EQ(ws.size(), 2u);                     // 深拷贝
    EXPECT_TRUE(other.contains(3));

    ws::list<int> target;
    target.push_back(99);
    target = std::move(other);                    // 移动赋值，覆盖原内容
    EXPECT_EQ(target.size(), 3u);
    EXPECT_FALSE(target.contains(99));
    EXPECT_TRUE(other.empty());
}

TEST(WsListTest, Contains)
{
    ws::list<int> ws;
    ws.push_back(10);
    ws.push_back(20);

    EXPECT_TRUE(ws.contains(10));
    EXPECT_TRUE(ws.contains(20));
    EXPECT_FALSE(ws.contains(30));
}
```

## 参数化测试：TEST_P 批量测 push_back

"用 0、1、2、3、10、100、1000 个元素分别推入"这类重复逻辑适合参数化测试：夹具继承 `::testing::TestWithParam<T>`，用 `GetParam()` 取当前参数，`INSTANTIATE_TEST_SUITE_P` 生成具体用例。每个参数是一条独立用例，失败时能看到具体是哪组数据。

```cpp
class WsVectorPushBackTest : public ::testing::TestWithParam<size_t> {
};

TEST_P(WsVectorPushBackTest, MatchesStdVector)
{
    const size_t count = GetParam();
    ws_vector<int> ws;
    std::vector<int> stdv;

    for (size_t i = 0; i < count; ++i) {
        const int value = static_cast<int>((i * 37 + 11) % 1001);
        stdv.push_back(value);
        ws.push_back(value);
    }

    ASSERT_EQ(ws.size(), stdv.size());
    EXPECT_TRUE(std::equal(ws.data(), ws.data() + ws.size(), stdv.begin()));
}

INSTANTIATE_TEST_SUITE_P(SizeCases, WsVectorPushBackTest,
                         ::testing::Values(0u, 1u, 2u, 3u, 10u, 100u, 1000u));
```

参数生成器除了 `Values(...)`，还有 `Range(begin, end)`、`ValuesIn(容器)`、`Bool()`，以及 `Combine(生成器1, 生成器2)` 做笛卡尔积。0 和 1 这种小边界最容易被漏掉，参数表里要包含。

## 死亡测试：验证断言真的会拦住非法调用

空容器调用 `pop_back()` 只有断言保护（`assert(!empty())`），普通用例没法验证——一触发进程就终止了。死亡测试在子进程中执行语句，检查进程是否按预期"死掉"，第二个参数匹配 stderr：

```cpp
#ifndef NDEBUG
TEST(WsVectorDeathTest, PopBackOnEmptyTriggersAssert)
{
    ws_vector<int> empty;
    ASSERT_DEATH(empty.pop_back(), "empty");
}
#endif
```

三点注意：断言在 `NDEBUG` 构建下会被编译掉，所以用例要包在 `#ifndef NDEBUG` 里；每个死亡测试都要 fork（POSIX）或重新执行进程（Windows），单条十毫秒起步，不适合大量使用；死亡测试必须在主线程调用，`GTest::gtest_main` 已经处理好了初始化。

## 运行：直接跑和交给 ctest

直接运行测试程序，输出完整的用例列表和统计：

```bash
./build/ws_test
```

真实输出节选（开头和结尾，中间是各套件逐条 `OK`）：

```text
[==========] Running 24 tests from 6 test suites.
[ RUN      ] WsVectorDeathTest.PopBackOnEmptyTriggersAssert
[       OK ] WsVectorDeathTest.PopBackOnEmptyTriggersAssert (14 ms)
...
[==========] 24 tests from 6 test suites ran. (13 ms total)
[  PASSED  ] 24 tests.
```

`gtest_discover_tests` 注册的用例可以直接用 ctest 跑，`--output-on-failure` 只在失败时打印详情：

```bash
ctest --test-dir build --output-on-failure
```

```text
24/24 Test #24: SizeCases/WsVectorPushBackTest.MatchesStdVector/1000 ...   Passed    0.01 sec

100% tests passed out of 24

Total Test time (real) =   0.19 sec
```

日常调试用过滤器只跑关心的用例：`./build/ws_test --gtest_filter='WsVectorTest.*'`；`--gtest_list_tests` 列出全部用例；`--gtest_repeat=10 --gtest_shuffle --gtest_break_on_failure` 组合起来适合排查偶发问题。

## 用 ASan 再跑一遍

单元测试验证逻辑，消毒器验证内存。配置一个新构建目录，把 `-fsanitize=address,undefined` 加进编译选项：

```bash
cmake -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug \
    -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer -g"
cmake --build build-asan -j 8
./build-asan/ws_test
```

```text
[==========] 24 tests from 6 test suites ran. (18 ms total)
[  PASSED  ] 24 tests.
```

如果容器有内存泄漏、重复释放、指针越界，ASan 会在对应用例处直接中止并打印调用栈。测试通过不等于内存没问题，这一步不能省。

## 覆盖率：看看还有哪条路没走

覆盖率不是目标，而是找盲区的手电筒。Clang 用 LLVM 的 profile 工具采集：

```bash
cmake -S . -B build-cov -DCMAKE_BUILD_TYPE=Debug \
    -DCMAKE_CXX_FLAGS="-fprofile-instr-generate -fcoverage-mapping"
cmake --build build-cov -j 8
./build-cov/ws_test

xcrun llvm-profdata merge -sparse default.profraw -o default.profdata
xcrun llvm-cov report ./build-cov/ws_test \
    -instr-profile=default.profdata ws_vector.h ws_list.hpp
```

真实报告（Linux 下去掉 `xcrun` 前缀）：

```text
Filename                      Regions    Missed Regions     Cover   Functions  Missed Functions  Executed       Lines      Missed Lines     Cover    Branches   Missed Branches     Cover
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
ws_vector.h                       112                30    73.21%          29                 0   100.00%         122                 9    92.62%          32                 6    81.25%
ws_list.hpp                        61                 3    95.08%          27                 0   100.00%          98                 4    95.92%          20                 5    75.00%
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
TOTAL                             173                33    80.92%          56                 0   100.00%         220                13    94.09%          52                11    78.85%
```

`ws_vector.h` 没走到的 9 行是 `resize`、`emplace_back`、带 count 的构造函数等本篇未覆盖的接口（由容器篇自己的测试负责）；`ws_list.hpp` 行覆盖率 95.92%，剩下的是防御分支。覆盖率的价值就在这里：它精确回答"哪些代码行从未被任何测试碰过"——比如只对拍 `size()` 的测试，覆盖率会立刻暴露 `operator[]`、`erase`、`at` 一行都没走到。用 GCC/lcov 的流程类似：`-DCMAKE_CXX_FLAGS="--coverage"` 构建运行后，`lcov --capture --directory build-cov --output-file coverage.info`，再用 `genhtml coverage.info --output-directory coverage_html` 生成网页报告。

## 踩坑与边界清单

1. **`EXPECT_*` 失败后代码继续跑**。后续语句依赖前面的结果时会连锁崩溃，掩盖真正原因。有依赖关系的检查前面用 `ASSERT_*`。
2. **自赋值不等于普通赋值**。`v = v` 走提前返回的短路分支，覆盖不到拷贝/移动赋值的主体逻辑，两类必须都测。
3. **测试数据要可复现**。直接用随机数生成数据，失败了也不知道是哪组输入；要随机压测可以，但种子必须打印或固定下来。
4. **死亡测试有成本和限制**。每个用例 fork/重启一次进程，几十毫秒起步；`NDEBUG` 下断言消失，用例要用 `#ifndef NDEBUG` 保护；不能跨线程调用。
5. **不要断言实现细节**。我们的容器承诺翻倍扩容，断言 `capacity < 2 * size` 合理；`std::vector` 的扩容策略标准没有规定，对标准库做同样断言就是埋雷。
6. **测试里的越界访问照样是 UB**。`v_std[pos]` 让 `pos` 取到 `100` 时就越界——测试代码本身也要过消毒器。

## 总结

- 自研容器的测试重点是生命周期、扩容边界、空容器行为、拷贝控制和越界检查；只对拍 `size()` 毫无意义，必须逐元素比较内容。
- 依赖管理交给 CMake + FetchContent，`GTest::gtest_main` 自带入口，`gtest_discover_tests` 把每个用例注册给 ctest；`#pragma comment(lib, ...)` 和 token 粘贴宏这样的写法要避开。
- 夹具 `TEST_F` 保证用例隔离，参数化 `TEST_P` 把边界值一次性铺开，死亡测试用于验证断言前置条件（仅 Debug 构建）。
- 逻辑用断言测，内存用 ASan 测，盲区用覆盖率找。本文 24 个用例在 Debug 和 ASan 下全部通过，`ws_vector.h`/`ws_list.hpp` 的行覆盖率分别达到 92.62% 和 95.92%。
