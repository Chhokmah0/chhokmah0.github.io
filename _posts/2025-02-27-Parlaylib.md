---
title: Parlaylib 的介绍和使用
date: 2025-2-13 12:00:00 +0800
categories: [笔记，并行计算]
tags: [并行计算]     # TAG names should always be lowercase
math: true
---
# Parlaylib

[Parlaylib](https://github.com/cmuparlay/parlaylib) 是一个在共享内存多核机器上的并行库。其包含一个顺序数据结构 `sequence`（类似于 `std::vector`）、许多并行算法、支持嵌套并行的工作窃取调度器、一个可扩展的内存分配器。[这里是其主页](https://cmuparlay.github.io/parlaylib/)。

Parlaylib 是一个头文件 only 的库，这意味着你可以很轻松的将其应用于自己的算法。由于我比较常用 xmake 作为项目的自动构建工具，因此我在这里给出 Parlaylib 在 xmake 里如何写成一个 package。

```lua
package("parlaylib")
    set_homepage("https://cmuparlay.github.io/parlaylib/")
    set_description("A Toolkit for Programming Parallel Algorithms on Shared-Memory Multicore Machines")
    set_license("MIT")
    set_urls("https://github.com/cmuparlay/parlaylib.git")
    add_deps("cmake")
    on_install(function (package)
        import("package.tools.cmake").install(package, {"-Dxxx=ON"})
    end)
package_end()
```

## 并行接口

### 常用并行函数

#### Binary forking (parallel_do)

```cpp
template <typename Lf, typename Rf>
void parallel_do(Lf&& left, Rf&& right, bool conservative = false)
```

`parallel_do` 接受两个函数对象并并行地求解它们。`left` 和 `right` 应该是不需要参数就能调用的函数，它们可能有返回值，但返回值会被忽略掉。

由于 Parlaylib 允许嵌套并行，所以我们可以用 `parallel_do` 写出下面的算法：

```cpp
template<typename Iterator>
auto sum(Iterator first, Iterator last) {
  if (first == last - 1) {
    return *first;
  }
  else {
    auto mid = first + std::distance(first, last) / 2;
    int left_sum, right_sum;
    parlay::par_do(
      [&]() { left_sum = sum(first, mid); },
      [&]() { right_sum = sum(mid, last); }
    )
    return left_sum + right_sum;
  }
}
```

#### Parallel-for loops (parallel_for)

```cpp
template <typename F>
void parallel_for(size_t start, size_t end, F&& f, long granularity = 0, bool conservative = false);
```

可执行函数 `f` 应该以一个 `size_t` 作为参数，`granularity` 和 `conservative` 的作用会在之后说明。

#### Blocked-for loops (blocked_for)

```cpp
template<typename F>
void blocked_for(size_t start, size_t end, size_t block_size, F&& f, bool conservative = false)
```

块并行类似于具有固定颗粒度（granularity）的 `parallel_for`，但是不是对范围的每个索引并行求值 `f`，而是将范围分成给定大小的连续块，并对每个块并行求 `f`。如果你有一个非常好的串行算法，那么使用 `blocked_for` “可能”会比 `parallel_for` 有着更好的性能。除了最后一个块，其余的块大小都是给定的 `block_size`。函数 `f` 会被传入三个 `size_t` 参数：分块的总数，块的开始位置，块的结束位置（不包含）。

```cpp
void copy_sequence(const parlay::sequence<int>& source, parlay::sequence<int>& dest) {
  // 这个算法*可能*比每个元素单独地并行复制更快
  parlay::blocked_for(0, source.size(), 512, [&](size_t i, size_t start, size_t end) {
    std::copy(source.begin() + start, source.begin() + end, dest.begin() + start);
  });
}
```

### 进一步的考量

#### 颗粒度（Granularity）

parallel-for 循环的粒度是对调度器的一个提示，即应该按顺序运行给定次数的循环迭代。例如，具有 100K 次迭代和粒度为 1000 的循环可以执行 100 个并行任务，每个任务负责以此计算 1000 次迭代。

需要注意颗粒度只是一个近似，所有的区间并不会恰好被分为大小相同的块。不同的调度器的运行方式可能并不一样。因此颗粒度应该被看作是一个优化提示，同时不应该与算法的正确性相关。

将颗粒度设置得过小会提高调度的开销，而将颗粒度设置得过大会降低算法的并行度。因此设计一个良好的颗粒度是很重要的。

当颗粒度为默认值（0）时，Parlay 的调度器会自动为循环估计一个颗粒度。调度器估计值通常非常好，接近最优值，因此很少需要手动调优粒度，除非 parallel-for 循环执行非常不规则和不可预测的工作量。

#### 保守调度

上述的并行算法都有一个参数 `conservative`，默认为 `false`。大部分情况下都可以忽略这个参数。只有当你在并行代码中使用了锁时才需要考虑它，打开后可以防止调度器按照可能死锁的顺序调度任务。这会导致一些性能损失，但该用的时候还是用上。

注意，锁有时可能被隐式获取，即使它们并不明显。例如，初始化函数作用域里的静态变量隐式地需要一个锁，因为它们的初始化被要求是线程安全的。因此，如果代码可能会在多个并行调用期间竞争初始化静态变量，则应该将这些调用的 `conservative` 设置为 `true`。

> 但是保守调度在底层上和普通调度的区别是什么？
{: .prompt-warning }

```cpp
void f(size_t i) {
    static size_t local_static = 0; // 局部 static 变量在控制流经过时才会被初始化
    local_static += i;
}

void not_obvious_mutex() {
    parlay::parallel_for(0, 10000000, f, 0, true);
}
```

## 详细内容

### 数据类型

#### sequence

```cpp
template <
    typename T,
    typename Allocator = parlay::allocator<T>,
    bool EnableSSO = std::is_same_v<T, char>
> class sequence
```

`sequence` 是一个类似于 `std::vector` 的数据类型。类似于 `std::string` 其实现了短字符串优化（Small String Optimization, SSO），并且可以通过将 `EnableSSO` 设置为 `true` 从而应用到任何类型上。Parlaylib 也提供了 `short_sequence<T, Allocator>` 作为 `sequence<T, Allocator, true>` 的别名。其内存分布大致为：

```cpp
struct {
  union {
    // Long sequence
    struct {
      void*            buffer     --->   size_t        capacity
      uint64_t         n : 48            value_type[]  elements
    }
    // Short sequence
    struct {
      unsigned char    buffer[15]
    }
  }

  // Only included when EnableSSO is true
  uint8_t              small_n : 7
  uint8_t              flag    : 1
}
```

一般推荐使用 `parlay::tabulate` 函数来构建 `sequence` 类型。

#### delayed_sequence

```cpp
template<
  typename T,
  typename V,
  typename F
> class delayed_sequence;
```

`delayed_sequence` 是一个懒函数式的序列，它会在访问的时候生成元素，而不是将元素存储在内存中。其是随机容器，但是并不包含缓存机制。其内部元素为：

```cpp
struct {
  size_t first, last;
  copyable_function_wrapper<F> f;
}
```

一般而言建议使用 `delayed_tabulate` 函数创建延迟序列容器。

```cpp
// 包含前 1000 个奇数的序列
auto seq = parlay::delayed_tabulate(1000, [](size_t i) {
  return 2*i + 1;
});
```

相较于 `sequence`，`delayed_sequence` 缺少许多函数，在功能上更接近于一个 `range`。

#### slice

```cpp
template <
    typename It,
    typename S
> struct slice
```

`slice` 是随机访问迭代器 `range` 的非所有视图。它表示一对迭代器，并允许方便地遍历和访问相应迭代器范围的元素。`It` 表示迭代器类型，`S` 表示迭代器最终的哨兵类型，通常来讲他们和 `It` 是相同的，但也不尽然。

切片的一个比较重要的函数是 `slice<It, It> cut(size_t ss, size_t ee) const`，可以返回从 `ss` 到 `ee` 的新切片。

一般使用工厂函数 `make_slice` 创建 `slice`，不过 `sequence` 中也有许多特殊的创建方式。

#### Phase-concurrent Deterministic Hashtable

```cpp
template <
    typename Hash
> class hashtable
```

该 hash 表允许并行插入，并行搜索，并行删除。但是不允许将插入、搜索、删除操作混合进行。散列策略必须提供“空”值的表示。整数值的默认哈希由 `hash_numeric<T>` 提供，这个策略使用 `-1` 作为空值，因此不能够用来存储 `-1`。

```cpp
parlay::hashtable<hash_numeric<int>> table;
table.insert(5);
auto val = table.find(5);  // Returns 5
table.deleteVal(5);
```

```cpp
// Example for hashing numeric values.
// T must be some integer type
template <class T>
struct hash_numeric {
  using eType = T;
  using kType = T;
  eType empty() { return -1; }
  kType getKey(eType v) { return v; }
  size_t hash(kType v) { return static_cast<size_t>(hash64(v)); }
  int cmp(kType v, kType b) { return (v > b) ? 1 : ((v == b) ? 0 : -1); }
  bool replaceQ(eType, eType) { return 0; }
  eType update(eType v, eType) { return v; }
  bool cas(eType* p, eType o, eType n) {
    // TODO: Make this use atomics properly. This is a quick
    // fix to get around the fact that the hashtable does
    // not use atomics. This will break for types that
    // do not inline perfectly inside a std::atomic (i.e.,
    // any type that the standard library chooses to lock)
    return std::atomic_compare_exchange_strong_explicit(
      reinterpret_cast<std::atomic<eType>*>(p), &o, n, std::memory_order_relaxed, std::memory_order_relaxed);
  }
};
```
