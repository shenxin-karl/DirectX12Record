# UploadBuffer

`UploadBuffer` 是在 `upload heap` 创建资源的线性分配器. 这个类的目的有三种类型数据到 GPU

* 上传 `ConstantBuffer`
* 上传 `VertexBuffer`
* 上传 `IndexBuffer`

在上传堆中创建大型资源是理想的选择, 使用 map 映射到地址上, 使用 `memcpy` 拷贝到 GPU

## 封装原理

`UploadBuffer` 为上传堆做一个简单的封装, 上传缓冲区被实现为一个线性分配器. 从内存页分配内存块. 如果内存 page 无法请求, 将一个新的内存 page  分配到激活列表中. 

![UploadBuffer1](D:\source\DirectX12Record\API封装\UploadBuffer\UploadBuffer1.png)

红色表示分配块, 绿色表示空闲块. 释放时直接释放整个 page. 然后 page 将被重置

## Upload Buffer Class

当不在需要 upload 缓冲区的数据时, 可以重用内存页. 这个 `UploadBuffer` 满足下面两个功能

* `Allocate`: 分配可用于将数据上载到 `GPU` 的内存块。

- `Reset`: 释放所有的内存, 重新使用

**一旦创建 page, 知道 `UploadBuffer` 析构, 否则不会被释放**, 上载缓冲区的目的是在每一帧重复使用它，以便在下一帧可能进行相同的分配.

```cc
#pragma once
#include <wrl.h>
#include <d3d12.h>
#include <memory>
#include <deque>

class UploadBuffer {
public:
    struct Allocation {
        void *CPU;
        D3D12_GPU_VIRTUAL_ADDRESS GPU;
    };
public:
    constexpr static kDefaultPageSize = 1024*1024*2;			// 2MB
    explicit UploadBuffer(size_t pageSize = kDefaultPageSize);	// 默认page的大小. 根据使用情况改变
    size_t getPageSize() { return _pageSize; }
    Allocation allocate(size_t sizeInBytes, size_t alignment);
    void reset();
private:
    size_t _pageSize;
};
```



