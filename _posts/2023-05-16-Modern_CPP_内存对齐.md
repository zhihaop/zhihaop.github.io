---
layout: post
title:  "Modern C++ ：内存对齐"
date:   2023-05-16 13:40:36 +0800
categories: jekyll update
---
## 1. 前言

C++11 引入几个内存对齐相关概念

- `alignof`
- `alignas`
- `std::aligned_storage`和`std::aligned_union`
- `std::align`

其中`std::aligned_storage`、`std::aligned_union`会在 C++23 中弃用，因此不作深入讨论。

### 1.1 什么是内存对齐

简而言之，就是某个变量地址必须是某个数的倍数，以方便 CPU 进行读取和访问。基本变量的内存布局都是按其`sizeof`大小进行对齐，也就是说`sizeof(x) = alignof(x)`。

#### 1.1.1 结构体的内存对齐

一个结构体的内存对齐，在默认情况下，取决于它的`field`中，内存对齐需求最大的哪个。

```cpp
// sizeof(Foo) = 24, alignof(Foo) = 8
struct Foo {
 char c;  // offsetof(Foo, c) = 0
 int  x;  // offsetof(Foo, x) = 4
 int  y;  // offsetof(Foo, y) = 8
 long z;  // offsetof(Foo, z) = 16
};
```

结构体中，每个`field`都 **刚好** 符合它的内存对齐需求。也就是说，结构体内部的对齐是一个 **贪心算法**。

#### 1.1.2 `alignof` & `alignas`

`alignof(x)`返回一个对象的`alignment`要求。但有时候，我们需要考虑 `cacheline`对齐 或`simd` 操作时，需要对象具有特殊的对齐方式。我们可以根据 `alignas`关键字改变一个对象的内存对齐。

```cpp
// sizeof(Foo) = 32, alignof(Foo) = 32
struct alignas(32) Foo {
 char c;  // offsetof(Foo, c) = 0
 int  x;  // offsetof(Foo, x) = 4
 int  y;  // offsetof(Foo, y) = 8
 long z;  // offsetof(Foo, z) = 16
};

void foo() {
 // alignment of buffer is 32.
 alignas(32) char buffer[128];
}
```

但是 `alignas`的使用具有一些限制：

- `alignas` 的值不能比对象默认的 `alignof`还小
- `alignas` 必须是 2 的指数幂

#### 1.1.3 `std::max_align_t`

有的时候我们想知道，在所有基本类型中，内存对齐的最大值是多少。C++11提供了`std::max_align_t`类型，它对应了内存对齐最大的基本类型，在大多数平台上是`long double`。

## 2. 内存管理与分配

### 2.1 `aligned_new`&`std::aligned_alloc`

一般来说，`new`和`std::malloc`返回的指针，其内存对齐为`alignof(std::max_align_t)`，有时候不能满足对象内存对齐的需求。`C++17`提供了支持自定义内存对齐的`new`和`malloc`，下面是一些例子。

```cpp
// alignment of p, q, buffer is 32.
int *p = new(std::align_val_t(32)) int;
int *q = new(std::align_val_t(32)) int[64];

void* buffer = std::aligned_alloc(std::align_val_t(32), 128);
```

### 2.2 `std::align`

我们有一块 `buffer`。我们希望从这块缓存中取出**符合内存对齐要求**的，**特定大小**的指针，就可以用`std::align`函数来帮助我们简化指针的获取。

```cpp
void* std::align(size_t alignment, size_t size, void*& ptr, size_t& space);

char buffer[1024];
void * head = buffer;
size_t remain = 1024;

// allocata an int.
int* p = reinterpret_cast<int*>(std::align(alignof(int), sizeof(int), head, remain));
```

### 2.3 `std::pmr::memory_resource`

`C++17`中添加了`多态内存分配器`，它也支持对齐的内存分配，其函数原型如下。其中 `allocate` 和 `deallocate`都需要提供分配指针的大小和对齐方式。

```cpp
[[nodiscard]] void* allocate(std::size_t bytes,
                             std::size_t alignment = alignof(std::max_align_t));

void deallocate(void* p, std::size_t bytes,
                std::size_t alignment = alignof(std::max_align_t));
```

### 2.4 高级专题

#### 2.4.1 空类的内存对齐

在`C++`标准中，一个结构体即使是空的，它也占有1字节的大小，目的是让每个实例都有不同的地址。但是当空类作为 **直接基类** 时，不需要为其分配空间。其内存对齐规则如下：

```cpp
// sizeof(empty_t) = 1, alignof(empty_t) = 1
struct empty_t {};

// sizeof(Foo) = 24, alignof(Foo) = 8
struct Foo : public empty_t {
 char c;  // offsetof(Foo, c) = 0
 int  x;  // offsetof(Foo, x) = 4
 int  y;  // offsetof(Foo, y) = 8
 long z;  // offsetof(Foo, z) = 16
};
```

但是如果一个类继承了两个空类，那么不能使用空基类优化，而且这个类也不属于空类。它的内存对齐规则如下：

```cpp
struct empty_t1 {};
struct empty_t2 {};

// sizeof(non_empty_t) = 2, alignof(non_empty_t) = 1
struct non_empty_t: public empty_t1, public empty_t2 {};
```

#### 2.4.2 虚表的内存对齐

虚表永远在一个类的头部，导致该类的内存对齐发生变化：

```cpp
// sizeof(Foo) = 24, alignof(Foo) = 8
struct Foo {
 char c; // offsetof(Foo, c) = 8
 int  x; // offsetof(Foo, x) = 12
 int  y; // offsetof(Foo, y) = 16

 virtual void foo() {}
};
```

#### 2.4.3 位域

位域的内存对齐行为通常根据编译器的不同而发生改变。在对内存要求严格的场景，应尽量不要使用位域。

#### 2.4.4 柔性数组

`C99`支持`柔性数组`，使用这个特性可以很方便的做不定长数组。包含柔性数组的结构体在内存对齐上与普通结构体相同，他们以一个例子说明。

```cpp
// sizeof(packet) = 8, alignof(packet) = 8
struct packet {
 int  size_;  // offsetof(packet, size_) = 0
 long data_[ ]; // offsetof(packet, data_) = 8
}

// malloc = sizeof(packet) + sizeof(T) * size
packet* new_packet(int sz) {
 packet* p = reinterpret_cast<packet*>(std::aligned_alloc(alignof(packet), sizeof(packet) + sizeof(long) * sz));
 p.size_ = sz;
 return p;
}
```

#### 2.4.5 `Eigen::aligned_allocator`

`Eigen`是一个`SIMD`矩阵运算库，因此对内存对齐有较高的要求。在`xmmintrin.h`中，定义了以下`SIMD`数据类型，其内存大小和对齐要求如下。其中，带`i`后缀是`int`，带`d`后缀是double，不带后缀为`float`。

|  | **指令集** | **大小** | **对齐** |
| --- | --- | --- | --- |
| `__m64` | `SSE` | 8 | 8 |
| `__m128`, `__m128i`, `__m128d` | `SSE2`,`SSE3`,`SSE4` | 16 | 16 |
| `__m256`,`__m256i`,`__m256d` | `AVX`, `AVX2` | 32 | 32 |
| `__m512`,`__m512i`,`__m512d` | `AVX512` | 64 | 64 |
