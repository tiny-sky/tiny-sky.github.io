---
title: std::tuple 解析
data: 2024-04-02
cover: /img/blog/blog0.jpg
categories:
- c++
tags:
- c++
---


# std::tuple

## 简介

std::tuple 是类似 pair 的模板。相对于 pair 只含有两个成员，每一个 tuple 实例都可以有不同的成员数目。

<!--more-->

## tuple 的创建与初始化

```cpp
std::tuple<T1, T2, TN> t1; //创建一个空的tuple对象（使用默认构造），它对应的元素分别是T1和T2...Tn类型。
std::tuple<T1, T2, TN> t2(v1, v2, ... TN); //创建一个tuple对象，它的两个元素分别是T1和T2 ...Tn类型。
```

这些方式属于直接构造 tuple 对象。同时也可以通过下面的方式创建 tuple 对象。

```cpp
auto t3 = std::tie(T1, T2, T3);  //返回左值引用的元组
auto t4 = std::make_tuple(T1,T2,T3); //创建一个拷贝对象

std::get<2>(t3); // 假定 T3 初始为 1
T3++;
std::get<2>(t3); // T3 变为 2

std::get<2>(t3); // 假定 T3 初始为 1
T3++;
std::get<2>(t3); // T3  还是 1
```

## tuple 元素的获取

```cpp
std::tie(T1,T2,T3) = t1; // std::tie 也可以解值，不过并非引用类型。
std::tie(std::ignore, T2, T3); // std::ignore 当作占位符
auto [T1,T2,T3] = t2; // C++17 引入的 structured binding 方法在解包时的表现与 std::tie() 类似。
auto [_1,_2,T3] = t3; // _1, _2 充当占位符
std::get<2>(t3); // 返回对应 tuple 位的引用
```

## 构造与解析的对比

```cpp
// 定义变量
std::string names{"Lihua"};
char ranks{'A'};
int score{2};
```

对比下面的例子:
```cpp
  int data;
  auto student0 = std::tie(names, ranks, score);
  std::tie(std::ignore, std::ignore, data) = student0;
  std::cout << "tie -> tie : ";
  if constexpr (std::is_same_v<int, decltype(data)>) {
    std::cout << "int" << std::endl;
  } else {
    std::cout << "int&" << std::endl;
  }
  // 输出 tie -> tie : int

  auto student1 = std::tie(names, ranks, score);
  auto [_3, _4, data2] = student1;
  std::cout << "tie -> [ ] : ";
  if constexpr (std::is_same_v<int, decltype(data2)>) {
    std::cout << "int" << std::endl;
  } else {
    std::cout << "int&" << std::endl;
  }
  // 输出 tie -> [ ] : int&
```
使用 std::tie 构造的 tuple 返回的左值引用，由于使用 std::tie 解包需要传入一个左值参数，所以解包后仍为左值，相当于赋值。

使用 structured binding 方法 auto [_1, _2, _3, ...] 在解包时,传入参数是一个未定义变量，所以解包后参数为右值。

再对比下面的例子
```cpp
  auto student2 = make_tuple(names, ranks, score);
  std::tie(std::ignore, std::ignore, data) = student0;
  auto [_1, _2, data1] = student2;
  std::cout << "make_tuple -> [ ] : ";
  if constexpr (std::is_same_v<int, decltype(data1)>) {
    std::cout << "int" << std::endl;
  } else {
    std::cout << "int&" << std::endl;
  }
  // 输出 make_tuple -> [ ] : int

  auto student3 = make_tuple(names, ranks, std::ref(score));
  auto [_5, _6, data3] = student3;
  std::cout << "make_tuple(std::ref) -> [ ] : ";
  if constexpr (std::is_same_v<int, decltype(data3)>) {
    std::cout << "int" << std::endl;
  } else {
    std::cout << "int&" << std::endl;
  }
  // 输出 make_tuple(std::ref) -> [ ] : int
```

使用 make_tuple 构造的 tuple 返回的是一个左值，且 std::tie 传入参数data为左值，所以解值也为左值。

最后一个例子使用 std::ref 来绑定引用进行传参。所以解出的值也为右值。

## 总结
- std::make_tuple() 以及 std::tie() 均可以构造一个 tuple，两者的区别在于前者构造复制，后者创建左值引用；
- std::get() 会解包出来固定位置的值，返回元组内部值的引用；
- std::tie() 会解包出来该元组的所有值，不需要的位置可以用占位符 std::ignore 代替，解包数据的数据类型是对应位置传入参数构造 tuple 的数据类型；
- C++17 引入的 structured binding 方法 auto [_1, _2, _3, ...] 在解包时的表现与 std::tie() 类似。