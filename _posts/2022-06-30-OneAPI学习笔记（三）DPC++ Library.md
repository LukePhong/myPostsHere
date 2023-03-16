---
layout:     post
title:      2022-06-30-OneAPI学习笔记（三）DPC++ Library
date:       2022-06-30
author:     lukephong
header-img: img/post-bg-sycl-structure.jpg
catalog: 	 true
tags:
    - oneAPI
    - sycl
    - 并行计算
---

# OneAPI学习笔记（三）DPC++ Library

by lukephong


## oneAPI DPC++ Library ***(oneDPL)***的容器、算法和使用

这才是真正的dpc++的东西，是intel的贡献，也是SYCL所不涵盖的地方。

oneDPL包括一系列可供并行执行的STL容器、算法和一些拓展，涵盖了我们熟悉的绝大多数c++标准中的内容。

比如你可以用以下方式，在GPU上为vector排序：

```c++
sycl::queue q(sycl::gpu_selector{});
std::sort(oneapi::dpl::execution::make_device_policy(q), v.begin(), v.end());
```

或使用特定的值填充一个vector

```c++
//该算法是使用Buffer的方式实现的
std::fill(oneapi::dpl::execution::make_device_policy(q), v.begin(), v.end(), 20);
```

### 使用的三种方式

概要：

1对于支持的C++标准API，可以通过直接include其头文件并使用namespace std即可使用。支持的API列表可以在[Tested Standard C++ APIs — oneAPI Libraries Documentation 1.0 documentation](https://docs.oneapi.io/versions/latest/onedpl/tested_standard_cpp_api.html?highlight=handler)找到。

2对于**Parallel STL**，它是C++17标准的一部分，使用方法可以参考[Get Started with Parallel STL (intel.com)](https://www.intel.com/content/www/us/en/developer/articles/guide/get-started-with-parallel-stl.html)。

3对于DPL拓展的API，使用方法可以在[Extension API — oneAPI Libraries Documentation 1.0 documentation](https://docs.oneapi.io/versions/latest/onedpl/extension_api.html)找到。

1、3需要的头文件都可以通过#include <oneapi/dpl/文件名>包含。对于1，如果直接包含如#include <utility>就需要使用namespace std，如果#include<oneapi/dpl/utility>，就需要使用namespace oneapi::dpl。2的使用需要安装其他依赖，这里就不展开了。

方式一：直接使用

可以像下面一样直接、连续的使用DPL算法。这种方式通过Buffer实现，会创建临时的Buffer，在算法执行前后，数据会会拷贝入临时的buffer中，之后再拷贝回来。

```c++
//==============================================================
// Copyright © 2020 Intel Corporation
//
// SPDX-License-Identifier: MIT
// =============================================================
//注意包含的头文件
#include <oneapi/dpl/algorithm>
#include <oneapi/dpl/execution>
#include<CL/sycl.hpp>
using namespace sycl;
using namespace oneapi::dpl::execution;

int main() {
  queue q;
  std::cout << "Device : " << q.get_device().get_info<info::device::name>() << "\n";
  std::vector<int> v{2,3,1,4};
    
  std::for_each(make_device_policy(q), v.begin(), v.end(), [](int &a){ a *= 2; });
  std::sort(make_device_policy(q), v.begin(), v.end());
    
  for(int i = 0; i < v.size(); i++) std::cout << v[i] << "\n";
  return 0;
}
```

方法二：通过Buffer调用

很明显，如果要连续使用DPL算法且操作的是同一块数据，那么像直接使用那样在device和host之间来回拷贝数据就没有意义了。Buffer则可以使需要处理的数据一直停留在device上，直到buffer销毁才拷贝回来。注意，为了让数据能够回来，我们必须把buffer销毁，也就是说，距离buffer实例化处最近的那一对{}是不能省略的。

需要更多包含一个头文件#include <oneapi/dpl/iterator>

然后你不需要写任何lambda表达式，直接传入buffer然后在代码块中调用相关算法即可。

```c++
//==============================================================
// Copyright © 2020 Intel Corporation
//
// SPDX-License-Identifier: MIT
// =============================================================

#include <oneapi/dpl/algorithm>
#include <oneapi/dpl/execution>
#include <oneapi/dpl/iterator>
#include <CL/sycl.hpp>
using namespace sycl;
using namespace oneapi::dpl::execution;
int main(){
  queue q;
  std::cout << "Device : " << q.get_device().get_info<info::device::name>() << "\n";
  std::vector<int> v{2,3,1,4};
    
  //# Create a buffer and use buffer iterators in Parallel STL algorithms
    //先用vector容器创建buffer。然后获得buffer开始和结束的指针（？）
  {
    buffer buf(v);
    auto buf_begin = oneapi::dpl::begin(buf);
    auto buf_end   = oneapi::dpl::end(buf);

    std::for_each(make_device_policy(q), buf_begin, buf_end, [](int &a){ a *= 3; });
    std::sort(make_device_policy(q), buf_begin, buf_end);
  }
    
  for(int i = 0; i < v.size(); i++) std::cout << v[i] << "\n";
  return 0;
}
```

方法三：USM法

通过USM使用DPL有两种方式：通过指针和通过allocators。

先介绍通过指针的方式。通过下面的例子相信能够很快的掌握其应用方法。

```c++
//==============================================================
// Copyright © 2020 Intel Corporation
//
// SPDX-License-Identifier: MIT
// =============================================================
//注意需要include的头文件
#include <oneapi/dpl/algorithm>
#include <oneapi/dpl/execution>
using namespace sycl;
using namespace oneapi::dpl::execution;
const int N = 4;

int main() {
  queue q;
  std::cout << "Device : " << q.get_device().get_info<info::device::name>() << "\n";
    
  //# USM allocation on device 使用正常的方式分配usm内存
  int* data = malloc_shared<int>(N, q);
    
  //# Parallel STL algorithm using USM pointer 直接调用所需函数，data+N显然是指向结尾的指针
  std::fill(make_device_policy(q), data, data + N, 20);
  q.wait();//注意需要等待执行结束
    
  for (int i = 0; i < N; i++) std::cout << data[i] << "\n";
  free(data, q);
  return 0;
}
```

上面分配的空间是一个数组，可是如果想要使用STL容器的话就只能使用Allocator了。

在这种情况下我们必须先创建allocator，然后用它来创建对应的vector，具体展示如下。经过测试，一个allocator可以用于在一个设备上分配不同的多个容器。

```c++
#include <oneapi/dpl/algorithm>
#include <oneapi/dpl/execution>

using namespace sycl;
using namespace oneapi::dpl::execution;

const int N = 4;

int main() {
  queue q;
  std::cout << "Device : " << q.get_device().get_info<info::device::name>() << "\n";
    
  //# USM allocator 创建usm_allocator，使用共享内存模式
  usm_allocator<int, usm::alloc::shared> alloc(q);
  //用usm_allocator创建vector
  std::vector<int, decltype(alloc)> v(N, alloc), v1(N, alloc);
    
  //# Parallel STL algorithm with USM allocator
  std::fill(make_device_policy(q), v.begin(), v.end(), 20);
  q.wait();
  std::fill(make_device_policy(q), v1.begin(), v1.end(), 30);
  q.wait();
    
  for (int i = 0; i < v.size(); i++) std::cout << v[i] << "\n";
  for (int i = 0; i < v1.size(); i++) std::cout << v1[i] << "\n";
  return 0;
}
```

# 未完待续
