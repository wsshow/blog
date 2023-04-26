---
title: 自定义动态数组（ws_vector）
date: 2021-07-18 16:18:30
author: ws
description: 使用基础元素构建C++动态数组
categories: 数据结构
tags: cpp
cover:
---

# 自定义动态数组（ws_vector）

## 1.声明局部变量

- _value:指针数组对象
- _size:数组元素个数
- _cap:数组容量

```cpp
template<typename ValueT>
class ws_vector
{
private:
	ValueT* _value;
	size_t _size{ 0 };
	size_t _cap{ 2 };
}
```

## 2.构造函数

```cpp
class ws_vector
{
public:
	ws_vector()
	{
		init();
	}

	ws_vector(size_t cap)
	{
		if (cap > 0) _cap = cap;
		init();
	}

	ws_vector(const ValueT& value, size_t size)
	{
		while (size--)
		{
			push_back(value);
		}
	}

	ws_vector(ws_vector& v) noexcept
	{
		*this = v;
	}

	void init()
	{
		_value = new DEBUG_CHECK_MEMORY_LEAKS ValueT[_cap];
	}
}
```

## 3.push_back

```cpp
/// <summary>
/// 在数组的最后添加一个数据
/// </summary>
/// <param name="value">需要添加的值</param>
void push_back(ValueT value)
{
    if (_size == _cap)
    {
    	_expand_cap(_cap << 1);
    }
    _value[_size++] = value;
}
```

## 4.pop_back

```cpp
/// <summary>
/// 去掉数组最后一个元素
/// </summary>
void pop_back()
{
    --_size;
}
```

## 5.resize

```cpp
/// <summary>
/// 重设容器大小
/// </summary>
/// <param name="size">容器大小</param>
void resize(size_t size)
{
    if (size < 0)
    {
        return;
    }

    if (size < _cap)
    {
        _size = size;
        return;
    }

    if (size >= _cap)
    {
        _expand_cap(size << 1);
        return;
    }
}
```

## 6.erase

```cpp
/// <summary>
/// 从容器擦除指定位置的元素
/// </summary>
/// <param name="index"></param>
void erase(size_t index)
{
    if (index >= _size) return;
    for (size_t i = index; i < _size; ++i)
    {
        _value[i] = _value[i + 1];
    }
}
```

## 7.swap

```cpp
/// <summary>
/// 将内容与 other 的交换
/// </summary>
/// <param name="other"></param>
void swap(ws_vector& other)
{
    ws_vector<ValueT> tmp = *this;
    *this = other;
    other = tmp;
}
```

## 8.clear

```cpp
/// <summary>
/// 从容器擦除所有元素。此调用后 size() 返回零
/// </summary>
void clear()
{
    if (_value)
    {
        delete[] _value;
    }
    _size = 0;
    _cap = 0;
}
```

## 9.reserve

```cpp
/// <summary>
/// 预分配集合容量
/// </summary>
/// <param name="cap">容量大小</param>
void reserve(size_t cap)
{
    if (cap <= _cap || cap > max_size())
    {
        return;
    }
    _expand_cap(cap);
}
```

## 10.contains

```cpp
/// <summary>
/// 集合中是否包含指定值
/// </summary>
/// <param name="value">需要检测的值</param>
/// <returns>若包含，则为true；反之，为false</returns>
bool contains(ValueT value)
{
    auto index = _size;
    while (index--)
    {
        if (value == _value[index])
        {
            return true;
        }
    }
    return false;
}
```

## 11.empty

```cpp
/// <summary>
/// 判断集合是否为空
/// </summary>
/// <returns>若为空，则为true；反之，为false</returns>
bool empty()
{
    return _size == 0;
}
```

## 12.size

```cpp
/// <summary>
/// 获取集合中元素个数
/// </summary>
/// <returns>元素个数</returns>
size_t size()
{
    return _size;
}
```

## 13.max_size

```cpp
/// <summary>
/// 获取集合的最大存储能力
/// </summary>
/// <returns>集合的最大存储能力</returns>
size_t max_size()
{
    return std::numeric_limits<size_t>::max();
}
```

## 14.capacity

```cpp
/// <summary>
/// 集合已分配空间的大小
/// </summary>
/// <returns>当前分配存储的容量</returns>
size_t capacity()
{
    return _cap;
}
```

## 15.at

```cpp
/// <summary>
/// 返回位于指定位置 index 的元素的引用，有边界检查
/// </summary>
/// <param name="index"></param>
/// <returns>index处元素的引用</returns>
ValueT& at(size_t index)
{
    if (index < 0 || index >= _size) throw(std::out_of_range("Parameter access out of range."));
    return _value[index];
}
```

## 16.operator[]

```cpp
/// <summary>
/// 返回位于指定位置 index 的元素的引用，无边界检查
/// </summary>
/// <param name="index"></param>
/// <returns>index处元素的引用</returns>
ValueT& operator[](size_t index)
{
    return _value[index];
}
```

## 17.front

```cpp
/// <summary>
/// 获取容器首元素的引用
/// </summary>
/// <returns>容器首元素的引用</returns>
ValueT& front()
{
    return _value[0];
}
```

## 18.back

```cpp
/// <summary>
/// 获取容器最后一个元素的引用
/// </summary>
/// <returns>容器最后一个元素的引用</returns>
ValueT& back()
{
    return _value[_size - 1];
}
```

## 19.data

```cpp
/// <summary>
/// 获取指向作为元素存储工作的底层数组的指针
/// </summary>
/// <returns>元素存储工作的底层数组的指针</returns>
ValueT* data()
{
    return _value;
}
```

## 20.operator=

```cpp
/// <summary>
/// 重载赋值运算符
/// </summary>
/// <param name="other">提供赋值数据的容器对象</param>
/// <returns></returns>
ws_vector& operator=(ws_vector& other)
{
    if (this == &other) return *this;

    delete[] _value;
    _size = other._size;
    _cap = other._cap;
    _value = new DEBUG_CHECK_MEMORY_LEAKS ValueT[_cap];

    for (size_t i = 0; i < _size; ++i)
    {
        _value[i] = other._value[i];
    }

    return *this;
}
```

## 21.析构函数

```cpp
~ws_vector()
{
    clear();
}
```

## 22.私有函数

```cpp
private:
	void _expand_cap(size_t cap)
	{
		ValueT* newValue = new DEBUG_CHECK_MEMORY_LEAKS ValueT[cap];
		for (size_t i = 0; i < _size; ++i)
		{
			newValue[i] = _value[i];
		}
		delete[] _value;
		_cap = cap;
		_value = newValue;
	}
```

