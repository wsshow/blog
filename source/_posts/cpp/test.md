---
title: 基于GTest的单元测试（ws_test）
date: 2021-08-06 23:18:30
author: ws
description: 基于GTest的单元测试
categories: 单元测试
tags: [cpp, gtest]
cover:
---

# 基于GTest的单元测试（ws_test）

## 完整代码

```cpp
#pragma once
#include <iostream>
#include <string>
#include <algorithm>
#include <vector>
#include <list>
#include <gtest/gtest.h>
#include "ws_vector.h"
#include "ws_list.h"

#define GTEST_LIB_PATH(path) "../GTest/"##path

#ifdef WS_TEST_DEBUG
#pragma comment(lib,GTEST_LIB_PATH("gtest_maind.lib"))
#pragma comment(lib,GTEST_LIB_PATH("gtestd.lib"))
#else
#pragma comment(lib,GTEST_LIB_PATH("gtest_main.lib"))
#pragma comment(lib,GTEST_LIB_PATH("gtest.lib"))
#endif

TEST(ListTest, push_back)
{
    std::list<int> lst_std;
    ws_list<int> lst_ws;
    for (size_t i = 0; i < 100; i++)
    {
        auto r = ws_random(0, 100);
        lst_std.push_back(r);
        lst_ws.push_back(r);
    }
    EXPECT_EQ(lst_std.size(), lst_ws.size());
}

TEST(VectorTest, push_back)
{
    std::vector<int> v_std;
    ws_vector<int> v_ws;
    for (size_t i = 0; i < 100; i++)
    {
        auto r = ws_random(0, 100);
        v_std.push_back(r);
        v_ws.push_back(r);
    }
    EXPECT_EQ(v_std.size(), v_ws.size());
}

TEST(VectorTest, RandomValues)
{
    std::vector<int> v_std;
    ws_vector<int> v_ws;
    for (size_t i = 0; i < 100; i++)
    {
        auto r = ws_random(0, 100);
        v_std.push_back(r);
        v_ws.push_back(r);
    }
    auto pos = ws_random(0, 100);
    EXPECT_EQ(v_std[pos], v_ws[pos]);
}

int test_all()
{
    printf("Running main() from %s\n", __FILE__);
    testing::InitGoogleTest();
    return RUN_ALL_TESTS();
}
```

