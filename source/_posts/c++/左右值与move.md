---
title: c++ 中左右值与 std::move 的解析
data: 2024-04-02
cover: /img/blog/blog1.jpg
categories:
- c++
tags:
- c++
---

# c++ 中左右值与 std::move 的解析

## 简介

在 c++ 中,对于拷贝我们经常会遇到以下的问题：
1. 十分影响性能的场景：
- 程序中存在大量的无用拷贝。
- 创建许多临时值，先将临时值进行拷贝，之后再将临时值释放。
2. 存在一些对象不能拷贝 ： unique_ptr

所以如果存在一种移动逻辑，将一个对象的所有权移动到另一个管理对象，这样就可以避免许多无用的拷贝以及无法拷贝，从而提升性能。

<!--more-->

## 左/右值

对移动的探讨，首先需要理解两个概念：左值、右值。

从代码逻辑上判断：左值就是 `=` 左边的值，即可以取地址，右值就是 `=` 右边的值，无法取地址。
```cpp
int a = 5; // a是左值,5是右值
B b = B(); // b是左值,B()是右值
``` 
## 左/右值引用

从名字上看左右值引用无非就是对应类型变量的别名，c++引入引用这种语法，本身是想降低对指针的使用难度，因为引用本身是通过指针来实现的，都是为了避免拷贝，直接访问对象。


### 左值引用

左值引用就是对左值的引用，给左值取别名。主要作用是避免对象拷贝。
```cpp
int a = 5;
int &ref_a = a; // 左值引用指向左值，编译通过
int &ref_a = 5; // 左值引用指向了右值，会编译失败
```

### 右值引用

右值引用就是对右值的引用，给右值取别名。主要作用是延长对象的生命周期。
```cpp
int &&ref_a_right = 5; // ok
 
int a = 5;
int &&ref_a_left = a; // 编译不过，右值引用不可以指向左值
 
ref_a_right = 6; // 右值引用的用途：可以修改右值
```

### 特例

既然左值引用可以指向左值，右值引用可以指向右值

那么左值引用是否可以指向右值？右值引用是否可以指向左值？

答案是可以的：
- const左值引用可以指向右值。
- 右值引用可以指向由 move(v) 修饰的左值。

1. const左值引用指向右值
```cpp
int &ref_a = 5;  // 编译失败
const int &ref_a = 5;  // 编译通过
```
具体的原因如下：
```
在 C++11标准产生之前，是没有右值引用这个概念的，当时如果想要一个类型既能接收左值也能接收右值的话，需要用const左值引用，比如标准容器的 push_back 接口：void push_back (const T& val)。
也就是说，如果const左值引用不能引用右值的话，有些接口就不好支持了。
另外，const左值引用，本意上是指向一个不被（该引用本身）它修改的内存区域，本质上这个引用变量本身也就是一个常量，至于这个内存区域对应一个全局变量、局部变量、xvalue，是不影响的。
```

2. 右值引用指向由 move(v) 修饰的左值
```cpp
int a = 5; // a是个左值
int &ref_left = a; // 左值引用指向左值
int &&ref_right = a; // 编译失败
int &&ref_right = std::move(a); // 通过std::move将左值转化为右值，可以被右值引用指向
 
cout << a; // 打印结果：5
```

我们发现 a 依然有值，其中的值并没有被"移动"

值的说明的是：std::move 本质上仅仅只是一个类型转换函数，唯一的功能是把左值强制转化为右值，让右值引用可以指向左值。而真正的移动逻辑是在调用对象的移动赋值/构造函数中。
```cpp
// FUNCTION TEMPLATE move
template <class _Ty>
_NODISCARD constexpr remove_reference_t<_Ty>&& move(_Ty&& _Arg) noexcept { // forward _Arg as movable
    return static_cast<remove_reference_t<_Ty>&&>(_Arg);
}

```

## 左值引用、右值引用本身是左值还是右值？

被声明出来的左、右值引用都是左值。 因为被声明出的左右值引用是有地址的，也位于等号左边。

```cpp
// 形参是个右值引用
void change(int&& right_value) {
    right_value = 8;
}
 
int main() {
    int a = 5; // a是个左值
    int &ref_a_left = a; // ref_a_left是个左值引用
    int &&ref_a_right = std::move(a); // ref_a_right是个右值引用
 
    change(a); // 编译不过，a是左值，change参数要求右值
    change(ref_a_left); // 编译不过，左值引用ref_a_left本身也是个左值
    change(ref_a_right); // 编译不过，右值引用ref_a_right本身也是个左值
     
    change(std::move(a)); // 编译通过
    change(std::move(ref_a_right)); // 编译通过
    change(std::move(ref_a_left)); // 编译通过
 
    change(5); // 当然可以直接接右值，编译通过
     
    cout << &a << ' ';
    cout << &ref_a_left << ' ';
    cout << &ref_a_right;
    // 打印这三个左值的地址，都是一样的
}
```
为什么 change(ref_a_right) 也会编译失败呢？ref_a_right不是个右值引用嘛？

值的说明的是：被声明出来的左、右值引用都是左值。 因为被声明出的左右值引用是有地址的，符合我们一开始对于左右值的定义(位于`=`左边)。

更具体的说：作为函数返回值的 && 是右值，直接声明出来的 && 是左值。

所以对于 ref_a_right 声明形式来看是一个右值引用，所以可以接受一个右值，不能接受左值。

但是本质上它等同于 `=`左边，是一个左值，所以在作为被使用对象(传入参数)是一个左值，故编译失败

### 小结

- 从传入参数来看：左右值都可以避免拷贝，不过右值引用可以直接接受左右值，而左值需要加上 const 才可以接受右值，在形式上有所限制

```cpp
void f(const int& n) {
    n += 1; // 编译失败，const左值引用不能修改指向变量
}

void f2(int && n) {
    n += 1; // ok
}

int main() {
    f(5);
    f2(5);
}
```
- 左值引用也可以指向右值(const&),右值引用也可以指向左值(move(v))

## 左值引用、右值引用的使用场景

1. 左值引用的使用场景

- 左值引用作为参数
```cpp
void func1(string s) {...}

void func2(const string& s) {...}

int main()
{
    std::string s1("Hello World!");
    func1(s1);  // 由于是传值传参且做的是深拷贝，代价较大
    func2(s1);  // 左值引用做参数减少了拷贝，提高了效率
    return 0;
}
```
- 左值引用作为返回值
```cpp
string s2("hello");

string operator+=(char ch)  // 传值返回存在拷贝且是深拷贝
string& operator+=(char ch)  // 左值引用做返回值没有拷贝，提高了效率

s2 += '!';
```

- 局限
```cpp
string operator+(const string& s, char ch) {
string ret(s);
ret.push_back(ch); // 不能返回局部变量的引用。局部变量会在函数返回后被销毁，此时局部变量的引用就会成为“无所指”的引用，程序会进入未知状态。
return ret;
}
```

既然左值引用也可以避免拷贝，那么什么时候应该用右值呢？

见下面的例子：
```cpp
class Array {
public:
    Array(int size) : size_(size) {
        data = new int[size_];
    }
     
    // 深拷贝构造
    Array(const Array& temp_array) {
        size_ = temp_array.size_;
        data_ = new int[size_];
        for (int i = 0; i < size_; i ++) {
            data_[i] = temp_array.data_[i];
        }
    }
     
    // 深拷贝赋值
    Array& operator=(const Array& temp_array) {
        delete[] data_;
 
        size_ = temp_array.size_;
        data_ = new int[size_];
        for (int i = 0; i < size_; i ++) {
            data_[i] = temp_array.data_[i];
        }
    }
 
    ~Array() {
        delete[] data_;
    }
 
public:
    int *data_;
    int size_;
};
```

尽管在传入参数这里使用左值引用减少了一次拷贝，但是内部需要实现深拷贝却无法避免，那么如果能将被拷贝者的数据移动过来，被拷贝者后边就不要了，就可以避免深拷贝了
```cpp
 // 移动构造函数，可以浅拷贝
    Array(const Array& temp_array, bool move) {
        data_ = temp_array.data_;
        size_ = temp_array.size_;
        // 为防止temp_array析构时delete data，提前置空其data_
        temp_array.data_ = nullptr; // temp_array是 const ,编译错误
    }
```
- 不够优雅，需要一个额外的参数(或者其他方式)。
- temp_array是个const左值引用，无法被修改。
- 如果改成非const：Array(Array& temp_array, bool move){...}，由于左值引用不能接右值，Array a = Array(Array(), true);这种调用方式就没法用了。

所以此时就可以使用右值引用来解决了这个问题。

```cpp
    Array& (const Array&& temp_array) {
        data_ = temp_array.data_;
        size_ = temp_array.size_;
        temp_array.data_ = nullptr; // 编译通过
    }
```
## 例子
```cpp
std::string str1 = "tiny-sky";
std::vector<std::string> vec;

vec.push_back(str1); // 传统方法，copy
vec.push_back(std::move(str1)); // 调用移动语义的push_back方法，避免拷贝，str1会失去原有值，变成空字符串
vec.emplace_back(std::move(str1)); // emplace_back效果相同，str1会失去原有值
vec.emplace_back("tiny-sky"); // 可以直接接右值

// 有些对象是 move-only 的
std::unique_ptr<A> ptr_a = std::make_unique<A>();

std::unique_ptr<A> ptr_b = ptr_a; // 编译失败
std::unique_ptr<A> ptr_b = std::move(ptr_a); // 编译通过
```

## 总结

- 左值即`=`左边的值，右值即`=`右边的值。
- 左值引用可以指向左值，const左值可以指向右值。
- 右值引用可以指向右值，右值引用可以指向move(v)的左值。
- 右值引用既可以是左值也可以是右值，区别在与作为函数返回值的 && 是右值，直接声明出来的 && 是左值。
- std::move 仅仅只做类型转换，具体的移动在对象的移动拷贝/构造逻辑中。
- 左值引用与右值引用都可以减少传入参数的拷贝，左值引用只有加上const才可以指向右值，但是const变量无法进行移动逻辑。

## 参考
[一文读懂C++右值引用和std::move](https://zhuanlan.zhihu.com/p/335994370)

[详解 C++ 左值、右值、左值引用以及右值引用](https://www.cnblogs.com/david-china/p/17080072.html)