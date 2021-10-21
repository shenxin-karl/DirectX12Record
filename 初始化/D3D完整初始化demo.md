# 完整的 D3D 初始化 demo

全局初始化

1. 用 `D3D12CreateDevice` 函数创建 `ID3D12Device` 接口实例
2. 创建一个 `ID3D12Fence` 对象，并查询描述符的大小
3. 检测用户设备对 `4X MSAA` 质量级别的支持情况
4. 依次创建命令队列、命令列表分配器和主命令列表
5. 描述并创建交换链
6. 创建应用程序所需的描述符堆

调整窗口后初始化(第一次也要初始化):

1. 调整后台缓冲区的大小，并为它创建渲染目标视图
2. 创建深度/模板缓冲区及与之关联的深度/模板视图
3. 设置视口 `viewport` 和裁剪矩形 `scissor rectangle`

**前置宏**

```cc
template<typename T>
T *_rigthValuePtr(T &&obj) {
    static thread_local T tempObj = T{};
    tempObj = std::forward<T>(obj);
    return &tempObj;
}
#define RVPtr(val) (_rigthValuePtr(val))			// return right object ptr

// if failed throw error exception
void _ThrowIfFailedImpl(const char *file, int line, HRESULT hr) {
    if (FAILED(hr))
		throw GraphicsException(file, line, hr);
}
#define ThrowIfFailed(val) (_ThrowIfFailedImpl(__FILE__, __LINE__, val))
```

```cc
// enabling d3d debugging
#if defined(DEBUG) || defined(_DEBUG)
{
    ComPtr<ID3D12Debug> debugController;
    ThrowIfFailed(D3D12GetDebugInterface(IID_PPV_ARGS(&debugController)));
    debugController->EnableDebugLayer();
}
#endif

// step1 create d3d device interface
// 创建 d3d 设备
/* calss member
	ComPtr<D3D12Device> 	d3dDevice_;
	ComPtr<IDXGIFactory1> 	dxgiFactory_;
*/

// try create hardward device
ComPtr<IDXGIFactory4> dxgiFactory;
CreateDXGIFactory1(IID_PPV_ARGS(&dxgiFactory));
ThrowIfFailed(CreateDXGIFactory1(IID_PPV_ARGS(dxgiFactory_)));
HRESULT hr = D3D12CreateDevice(nullptr, D3D_FEATURE_LEVEL_11_0, IID_PPV_ARGS(&d3dDevice_));
if (FAILED(hr)) {		// fallback to wapp device
    ComPtr<IDXGIAdapter> pWarpAdapter;
    ThrowIfFailed(dxgiFactory->EnumWarpAdapter(IID_PPV_ARGS(&pWarpAdapter)));
    ThrowIfFailed(D3D12CreateDevice(
        pWarpAdapter.Get(), 
        D3D_FEATURE_LEVEL_11_0, 
        IID_PPV_ARGS(&d3dDevice_)
    ));
}

// step2 create GPU fence and get RTV DSV cvbUav desciptor size
// 创建一个 ID3D12Fence 对象，并查询描述符的大小
/* class member
	ComPtr<ID3D12Fence>				fence_;
	int 							rtvDescriptorSize_;
	int 							dsvDescriptorSize_;
	int 							cbvUavDescriptorSize_;
*/
ThrowIfFailed(d3dDevice_->CreateFence(0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&fence_)));
rtvDescriptorSize_ = d3dDevice_->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_RTV);
dsvDescriptorSize_ = d3dDevice_->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_DSV);
cbvUavDescriptorSize_ = 
    d3dDevice_->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);

// step3 check 4x msaa support
// 检查 4xmsaa 的支持情况
/* class member
	int 		msaaQualityLevel_;
	DXGI_FORAMT backBufferFormat_;		// 默认的后台缓区纹理类型
*/
D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS msQualityLevels;
msQualityLevels.Format = backBufferFormat_;
msQualityLevels.SampleCount = 4;
msQualityLevels.Flags = D3D12_MULTISAMPLE_QUALITY_LEVELS_FLAG_NONE;
msQualityLevels.NumQualityLevels = 0;
ThrowIfFailed(d3dDevice_->CheckFeatureSupport(
	D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS
    &msQualityLevels,
    sizeof(msQualityLevels)
));
msaaQualityLevel_ = msaaQualityLevel_.NumQualityLevels;
assert(msaaQualityLevel_ > 0);


```

