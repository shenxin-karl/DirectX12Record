# 初始 Direct3D

初始化步骤:

1. 用 `D3D12CreateDevice` 函数创建 `ID3D12Device` 接口实例
2. 创建一个 `ID3D12Fence` 对象，并查询描述符的大小
3. 检测用户设备对 `4X MSAA` 质量级别的支持情况
4. 依次创建命令队列、命令列表分配器和主命令列表
5. 描述并创建交换链
6. 创建应用程序所需的描述符堆
7. 调整后台缓冲区的大小，并为它创建渲染目标视图
8. 创建深度/模板缓冲区及与之关联的深度/模板视图
10. 设置视口 `viewport` 和裁剪矩形 `scissor rectangle`

## 创建设备

一个设备代表着一个显示适配器. 用下面的函数创建 Direct3D 12 设备

```cc
/* @brief:						创建一个 Direct3d 设备
 * @param pAdapter:				指定显示适配,如果为空使用主显示器适配器
 * @param MininumFeatureLevel:	应用程序所需要的最小等级我们使用(D3D_FEATURE_LEVEL_11_0)
 * @param riid:					创建的 ID3D12Device 接口的 COM ID
 * @param ppDevice:				返回创建的接口
 */
HRESULT WINAPI D3D12CreateDerice(
	IUnknow *pAdapter,
    D3D_FEATURE_LEVELE MininumFeatureLevel, 
    REFIID riid,
    void **ppDevice
);
```

调用示例:

```cc
#if defined(DEBUG) || defined(_DEBUG)		// 开启D3D12 Debug调试
{	
    ComPtr<ID3D12Debug> debugController;
    ThrowIfFailed(D3D12GetDebugInterface(IID_PPV_ARGS(&debugController)));
    debugController->EnableDebugLayer();		
}
#endif

// 使用主显示器创建 Device
// WRL::ComPtr<IDXGIFactory1>	dxgiFactory_;
//	WRL::ComPtr<ID3D12Device>	d3dDevice_;
ThrowIfFailed(CreateDXGIFactory1(IID_PPV_ARGS(&mdxgiFactory_)));
HRESULT hr = D3D12CreateDevice(
	nullptr,
    D3D_FEATRUE_LEVEL_11_0,
    IID_PPV_ARGS(&md3dDevice)
);
// 创建 DXGI 工厂
ComPtr<IDXGIFactory4> dxgiFactory;
CreateDXGIFactory1(IID_PPV_ARGS(&dxgiFactory));

// 创建失败回退到 WRAP 设备
if (FAILED(hr)) {
    ComPtr<IDXGIAdapter> pWarpAdapter;
    ThrowIfFailed(dxgiFactory->EnumWarpAdapter(IID_PPV_ARGS(&pWarpAdapter)));
    ThrowIfFailed(D3D12CreateDevice(
    	pWarpAdapter.Get(),
        D3D_FEATURE_LEVEL_11_0,
        IID_PPV_ARGS(&d3dDevice)
    ));
}
```

## 创建围栏并获取描述符的大小

创建好 `Device` 后, 便可以为 `CPU/GPU` 的同步创建围栏了. 

```cc
// 创建围栏 Fence
ThrowIfFailed(d3dDevice->CreateFence(
	0, D3D12_FENCE_FLAG_NONE, IID_PPV_ARGS(&fence_)
));
// 查询 RTV 大小
rtvDescriptorSize_ = d3dDevice_->GetDescriptorHandleIncrementSize(
	D3D12_DESCRIPTOR_HEAP_TYPE_RTV);
// 查询 DSV 大小
dsvDescriptorSize_ = d3dDevice_->GetDescriptorHandleIncrementSize(
	D3D12_DESCRIPTOR_HEAP_TYPE_DSV);
// 查询 CBV UAV 大小
cbvUavDescriptorSize_ = d3dDevice_->GetDescriptorHandleIncrementSize(
	D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);
```

## 检测对 4XMSAA 质量级别的支持

```cc
D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS msQualityLevels;
msQualityLevels.Foramt = mBackBufferFormat;	// 传递帧缓冲纹理类型
msQualityLevels.SampleCount = 4;
msQualityLevels.Flags = D3D12_MULTISAMPLE_QUALITY_LEVELS_FLAG_NONE;
msQualityLevels.NumQualityLevels = 0;
ThrowIfFailed(d3dDevice_->CheckFeatureSupport(
	D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS,
    &msQualityLevels,
    sizeof(msQualityLevels)
));
msaaQuality_ = msQualityLevels.NumQualityLevels;
assert(msaaQuality_ > 0 && "Unexpected MSAA quality level");
```

## 创建命令队列和命令列表

接口

* `ID3D12CommandQueue` 接口表示命令队列. 
* `ID3D12CommandAllocator` 接口表示命令分配器
* `ID3D12CommandList` 接口表示命令列表

```cc
// 接口指针
ComPtr<ID3D12CommandQueue> 	  		commandQueue_;
ComPtr<ID3D12ComandAllocator> 		directCmdListAllo_;
ComPtr<I3D312GraphicsCommandList>	commandList;
void D3DApp::createCommandObjects() {
    // 创建命令队列
	D3D12_COMMAND_QUEUE_DESC queueDesc = {}; 
    queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
	queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
	ThrowIfFailed(d3dDevice_->CreateCommandQueue(
        &queueDesc, IID_PPV_ARGS(&commandQueue_)
    ));
    // 创建队列分配器
    ThrowIfFailed(d3dDevice_->CreateCommandAllocator(
    	D3D12_COMMAND_LIST_TYPE_DIRECT,
        IID_PPV_ARGS(&directCmdListAlloc_)
    ));
    // 创建命令列表
    
}
```

## 描述并创建交换链

需要填写一份 `DXGI_SWAP_CHAIN_DESC` 结构体实例. 

```cc
typedef struct DXGI_SWAP_CHAIN_DESC {
    DXGI_MODE_DESC 		BufferDesc;
    DXGI_SAMPLE_DESC 	SampleDesc;
    DXGI_USAGE			BufferUsage;
    UINT 				BufferCount;
    HWND				OutPutWindow;
    BOOL				Windowed;
    DIGI_SWAP_EFFECT 	SwapEffect;
    UINT				Flags;
} DXGI_SWAP_CHAIN_DESC;
```

其中 `DXGI_MODE_DESC` 是另外一份结构体

```cc
typedef struct DXGI_MODE_DESC {
	UINT						Width;				// 宽
    UINT						Height;				// 高
    DXGI_RATIONAL 				RefreshRate;		// 刷新率
    DXGI_FORMAT					Format;				// 缓冲区格式
    DXGI_MODE_SCANLINE_ORDER	ScanlineOrder;		// 扫描线顺序
    DXGI_MODE_SCALING			Scaling;			// 图像如何相对于屏幕拉伸
} DXGI_MODE_DESC;
```

```cc
/*
 * BufferDesc:	描述待创建的后台缓冲区的属性这里我们仅关注, 宽度高度和像素格式属性
 * SampleDesc:	多重采样的质量级别,每个像素的采样次数. 单次采样(次数1,质量0)
 * BufferUsage:	由于我们需要将数据渲染到后台缓冲区,
 * 				指定为, DXGI_USAGE_RENDER_TARGET_OUTPUT
 * BufferCount:	交换链中所有的缓冲区数量. 我们指定为2
 * OutputWindow:渲染窗口的句柄
 * Windowed:	true窗口模式, false全屏模式
 * SwapEffect:	指定为 DXGI_SWAP_EFFECT_FLIP_DISCARD
 * Flag:		可选标志: 
 *				DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH 切换为全屏模式时, 
 				选择最合适的显示模式.
```

描述完交换链后, 我们使用 `IDXGIFactory::CreateSwapChain` 来创建它

```cc
/* @brief:				创建SwapChain
 * param pDevice:		指向ID3D12CommandQueue的指针
 * param pDesc:			指向描述交换链结构体指针
 * param ppSwapChain:	返回值
 */
HRESULT IDXGIFactory::CreateSwapChain(
	IUnknow *pDevice,
    DXGI_SWAP_CHAIN_DESC *pDesc,
    IDXGISwapChain 	**ppSwapChain
);
```

示例:

```cc
ComPtr<IDXGISwapChain> swapChain_;		// 类型定义
void Graphics::createSwapChain() {
	swapChain_.Reset();
	DXGI_SWAP_CHAIN_DESC sd;
	sd.BufferDesc.Width = D3DApp::instance()->getWindow()->getWidth();
	sd.BufferDesc.Width = D3DApp::instance()->getWindow()->getHeight();
	sd.BufferDesc.RefreshRate.Numerator = 60;
	sd.BufferDesc.RefreshRate.Denominator = 1;
	sd.BufferDesc.Format = DXGI_FORMAT_R32G32B32A32_FLOAT;
	sd.BufferDesc.ScanlineOrdering = DXGI_MODE_SCANLINE_ORDER_UNSPECIFIED;
	sd.BufferDesc.Scaling = DXGI_MODE_SCALING_UNSPECIFIED;
	sd.SampleDesc.Count = msaaState_ ? 4 : 1;
	sd.SampleDesc.Quality = msaaState_ ? (msaaQualityLevel_-1) : 0;
	sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
	sd.BufferCount = 2;
	sd.OutputWindow = D3DApp::instance()->getWindow()->getHWND();
	sd.Windowed = true;
	sd.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;
	sd.Flags = DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH;
	ThrowIfFailed(dxgiFactory_->CreateSwapChain(
		commandQueue_.Get(),
		&sd,
		swapChain_.GetAddressOf()
	));
}
```

## 创建描述堆

我们需要对比创建描述符堆来存储程序中所需要的的描述符/视图. `Direct3D 12` 以
`ID3D12DescriptorHeap` 接口来表示描述符堆. 并用 `ID3D12Device::CreateDesciptorHeap` 来创建它.

将为交换链中的 `SwapChainBufferCount` 个用于渲染的数据的缓冲区创建对应渲染目标视图
`Render Target View RTV`. 并为用于深度测试 `depth test` 的深度/模板缓冲区创建一个**深度/模板视图(`Depth Stencil View DSV`)**

```CC
ComPtr<ID3D12DesciptorHeap> rtvHeap;
ComPtr<ID3D12DesciptorHeap>	dsvHeap;
int currBackBuffer_ = 0;						// 记录当前的后台缓冲索引
static constexpr int kSwapChainBufferCount = 2;	// 交换链的数量
void D3DApp::CreateRtvAndDsvDesciptorHeaps() {
   	D3D12_DESCRIPTOR_HEAP_DESC rtvHeapDesc;
	rtvHeapDesc.NumDescriptors = kSwapChainCount;		// 需要2个视图
	rtvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
	rtvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
	rtvHeapDesc.NodeMask = 0;
	ThrowIfFailed(d3dDevice_->CreateDescriptorHeap(
		&rtvHeapDesc,
		IID_PPV_ARGS(&rtvHeap)
	));

	D3D12_DESCRIPTOR_HEAP_DESC dsvHeapDesc;
	dsvHeapDesc.NumDescriptors = 1;						// 只需要一个视图
	dsvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_DSV;
	dsvHeapDesc.Flags = D3D12_DESCRIPTOR_HEAP_FLAG_NONE;
	dsvHeapDesc.NodeMask = 0;
	ThrowIfFailed(d3dDevice_->CreateDescriptorHeap(
		&dsvHeapDesc,
		IID_PPV_ARGS(&dsvHeap)
	));
}
```

创建描述符堆后. 还要能访问存储的描述符. 

**获得 `RTV`**

我们可以通过 `ID3D12DescriptorHeap::GetDescriptorHandleForHeapStart` 获得句柄; 

```cc
D3D12_CPU_DESCRIPTOR_HANDLE D3DApp::currentBackBufferView() const {
    return CD3DX12_CPU_DESCRIPTOR_HANDLE(
    	rtvHeap->GetDescriptorHandleForHeapStart(),	// 首个句柄
        currBackBuffer_,							// 偏移到后台缓冲区描述符
        rtvDescriptorSize							// 描述符占用的大小
    );
}
```

**获得 `DSV`**

```cc
D3D12_CPU_DESCRIPTOR_HANDLE D3DApp::DepthStencilView() const {
    return dsvHeap->GetDescriptorHandleForHeapStart();
}
```

## 创建深度/模板缓冲区及其视图

纹理是一种GPU资源, 因此我们需要通过填写 `D3D12_RESOURCE_DESC` 结构体描述纹理资源

```cc
typedef struct D3D12_RESOURCE_DESC {
    D3D12_RESOURCE_DIMENSION 	Dimension;
    UINT64						Alignment;		
    UINT64						Width;
    UINT64						Height;
    UINT64						DepthOrArraySize;
    UINT64						MipLevels;
    DXGI_FORMAT					Format;
    DXGI_SAMPLE_DESC			SampleDesc;
    D3D12_TEXTURE_LAYOUT		Layout;
    D3D12_RESOURCE_MTSC_FLAG	MiscFlags;
} D3D12_RESOURCE_DESC;
```

```cc
/*
 * Dimension:			纹理的类型
 * Alignment:			对齐设置为 0
 * Width:				纹理的宽度
 * Height:				纹理的高度
 * DepthOrArraySize:	以纹素单位来表示纹理深度, 设置为 1
 * MipLevels:			mipmap的层级数量.深度和模板只能有一个mipmap等级
 * Format:				DXGI_FORMAT 枚举成员之一. 深度模板类型使用 DXGI_FORMAT_R24G8_TYPELESS
 * SampleDesc:			多重采样的质量级别每个像素的采样次数
 * Layout:				暂时不用考虑这个问题.暂时指定为 D3D12_TEXTURE_LAYOUT_UNKNOWN
 * Flags:				与资源有关的杂项标志;对于深度/模板缓冲区资源来说需要指定
 						D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL
```

GPU资源都存储在堆中. 其本质是具有特定属性的显存块. 

使用 `ID3D12Device::CreateCommittedResource` 创建

```CC
typedef struct D3D12_HEAP_PROPERTIES {
    D3D12_HEAP_TYPE				Type;
    D3D12_CPU_PAGE_PROPERTY		CPUPageProperty;
    D3D12_MEMORY_POOL			MemoryPoolPreference;
    UINT						CreationNodeMask;
    UINT						VisibleNodeMask;
} D3D12_HEAP_PROPERTIES;

typedef struct D3D12_CLEAR_VALUE {
    DXGI_FORMAT		Format;
    union {
        float						Color[4];
        D3D12_DEPTH_STENCIL_VALUE	DepthStencil;
    };
} D3D12_CLEAR_VALUE;

/* @brief:						创建资源
 * @parma pHeapProperties:		D3D12_HEAP_PROPERTIES 指针
 * @parma HeapFlags:			通常使用 D3D12_HEAP_FLAG_NONE
 * @param pDesc:				指向 D3D12_RESOURCE_DESC
 * @param InitialResourceState:	资源的初始状态,通常设置为D3D12_RESOURCE_STATE_COMMON
 * @param pOptimizedClearValue:	清理资源的值, 不使用可以传递 nullptr
 * @param riidResource:			COMID
 * @param ppvResource:			返回值
 */
HRESULT ID3D12Device::CreateCommittedResource(
    const D3D12_HEAP_PROPERTIES *pHeapProperties
	D3D12_HEAP_FLAGS HeapFlags,
	const D3D12_RESOURCE_DESC *pDesc,
    D3D12_RESOURCE_STATES InitialResourceState,
    const D3D12_CLEAR_VALUE *pOptimizedClearValue,
    REFIID riidResource,
    void **ppvResource
);
```

**创建深度/模板视图示例**

在使用深度/模板缓冲区之前, 一定要创建相关的视图绑定到流水线上

```cc
ComPtr<ID3D12Resource>	depthStencilBuffer_;
DXGI_FORMAT		depthStencilFormat_ = DXGI_FORMAT_D24_UNORM_S8_UINT; //默认

D3D12_RESOURCE_DESC depthStencilDesc;
depthStencilDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
depthStencilDesc.Alignment = 0;
depthStencilDesc.Width = D3DApp::instance()->getWindow()->getWidth();
depthStencilDesc.Height = D3DApp::instance()->getWindow()->getHeight();
depthStencilDesc.DepthOrArraySize = 1;
depthStencilDesc.MipLevels = 1;
depthStencilDesc.Format = depthStencilFormat_;
depthStencilDesc.SampleDesc.Count = msaaState_ ? 4 : 1;
depthStencilDesc.SampleDesc.Quality = msaaState_ ? (msaaQualityLevel_-1) : 0;
depthStencilDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
depthStencilDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;

D3D12_CLEAR_VALUE optClear;
optClear.Format = depthStencilFormat_;
optClear.DepthStencil.Depth = 1.0f;
optClear.DepthStencil.Stencil = 0;
ThrowIfFailed(d3dDevice_->CreateCommittedResource(
    &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
    D3D12_HEAP_FLAG_NONE,
    &depthStencilDesc,
    D3D12_RESOURCE_STATE_COMMON,
    &optClear,
    IID_PPV_ARGS(&depthStencilBuffer_)
));

// 为第一层mipmap创建一个视图
d3dDevice_->CreateDepthStencilView(
    depthStencilBuffer_.Get(),
    nullptr,
    depthStencilView()
);

// 将资源从初始状态转换为深度缓冲区
auto resourceBarrierObj = CD3DX12_RESOURCE_BARRIER::Transition(
    depthStencilBuffer_.Get(),
    D3D12_RESOURCE_STATE_COMMON,
    D3D12_RESOURCE_STATE_DEPTH_WRITE
);
commandList_->ResourceBarrier(1, &resourceBarrierObj);

```

## 设置视口

我们能够绘制到窗口的某个矩形上而不是整个窗口, 这个我们称为**视口**. 视口使用下面的结构描述

```cc
typedef struct D3D12_VIEWPORT {
    FLOAT	TopLeftX;
    FLOAT	TopLeftY;
    FLOAT	Width;
    FLOAT	Height;
    FLOAT	MinDepth;		
    FLOAT	MaxDepth;
} D3D12_VIEWPORT;

// MinDepth 和 MaxDepth 负责将 0~1之间的深度转换为 [MinDepth, MaxDepth].
```

**实例**

```cc
D3D12_VIEWPORT vp;
vp.TopLeftX = 0.f;
vp.TopLeftX = 0.f;
vp.Width = D3DApp::instace()->getWindow()->getWidth();
vp.Height = D3DApp::instance()->getWindow()->getHeight();
vp.MinDepth = 0.f;
vp.MaxDepth = 1.f;
CommonList_->RSSetViewports(1, &vp);
```

> 命令列表被重置以后, 同时视口也被重置

## 设置裁剪矩形

我们可以定义一个裁剪矩形. 在此矩形外面的像素都被剔除. 裁剪矩形由 `D3D12_RECT` 结构体描述

```cc
typedef struct tagRECT {
	LONG	left;
    LONG	top;
    LONG	right;
    LONG	bottom;
} RECT;
```

在 `Direct3D` 中, 要使用 `ID3D12GrahicsCommandList::RSSetScissoorRects` 方法来设置裁剪矩形

```cc
scissorRect_ = { 0, 0, width, height };
commandList_->RSSetScissoorRects(1, &scissorRect_);
```

**Note:**

> 不能为同一个渲染目标指定多个裁剪矩形; 裁剪矩形需要随着命令列表的重置而重置	CreateCommittedResourceCD3D12_RESOURCE_DESC
