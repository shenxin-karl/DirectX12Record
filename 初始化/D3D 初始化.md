# D3D 初始化

初始化分为两部分:

*   直接初始化
*   窗口更改初始化

## 直接初始化部分

### 开启调试层

```cc
#if defined(DEBUG) || defined(_DEBUG) 
{	// Enable the D3D12 debug layer.
	ComPtr<ID3D12Debug> debugController;
	ThrowIfFailed(D3D12GetDebugInterface(IID_PPV_ARGS(&debugController)));
	debugController->EnableDebugLayer();
}
#endif
```

### 创建设备

使用到的方法

```cc
/* @pAdapter:				指定显示适配器,传递空指针使用主显示器
 * @MininumFeatureLevel:	最小支持的 D3D 等级
 * @riid:					COM ID
 * @ppDevice:				返回值
 */
HRESULT WINAPI D3D12CreateDevice(
	IUnknow 			  *pAdapter,			
    D3D12_FEATURE_LEVEL 	MininumFeatureLevel,
    REFIID					riid,
    void 				  **ppDevice
);
```

调用示例

```cc
ComPtr<IDXGIFactory4>		dxgiFactory_;
ComPtr<ID3D12Device>		d3dDevice_;

ThrowIfFailed(CreateDXGIFactory1(IID_PPV_ARGS(dxgiFactory_));
HRESULT hr = D3D12CreateDevice(nullptr, D3D_FEATURE_LEVEL_11_0, IID_PPV_ARGS(&d3dDevice_));
if (FAILED(hr)) {		// 创建失败.使用 WARP 设备
    ComPtr<IDXGIAdapter>  pWarpAdapter;
    ThrowIfFailed(dxgiFactory_->EnumWarpAdapter(IID_PPV_ARGS(&pWarpAdapter)));
    ThrowIfFailed(pWarpAdapter.Get(), D3D12_FEATURE_11_0, IID_PPV_ARGS(&d3dDevice_));
}
```

### 创建围栏并获取描述符大小

```cc
ComPtr<ID3D12Fence> fence_;
UINT rtvDescriptorSize_ = 0;
UINT dsvDescriptorSize_ = 0;
UINT cbvSrvUavDescriptorSize_ = 0;

// 创建围栏
ThrowIfFailed(d3dDevice_->CreateFence(
    0, 
    D3D12_FENCE_FLAG_NONE,
    IID_PPV_ARGS(&fence_)
));

// 获取描述符大小
rtvDescriptorSize_ = md3dDevice->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_RTV);
dsvDescriptorSize_ = md3dDevice->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_DSV);
cbvSrvUavDescriptorSize_ 
    = d3dDevice_->GetDescriptorHandleIncrementSize(D3D12_DESCRIPTOR_HEAP_TYPE_CBV_SRV_UAV);
```

### 检测对4X MSAA质量级别的支持

**检测结构体**

```cc
typedef struct D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS {
  DXGI_FORMAT                           	Format;				// RTV纹理格式
  UINT                                  	SampleCount;		// 需要采样数,一般为4
  D3D12_MULTISAMPLE_QUALITY_LEVEL_FLAGS 	Flags;				// 标志位
  UINT                                  	NumQualityLevels;	// 返回支持的采样数
} D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS;
```

**调用示例**

```cc
UINT msaaQuality_;

D3D12_FEATURE_DATA_MULTISAMPLE_QUALITY_LEVELS msQualityLevels;
msQualityLevels.Format = mBackBufferFormat;	// DXGI_FORMAT_R8G8B8A8_UNORM
msQualityLevels.SampleCount = 4;
msQualityLevels.Flags = D3D12_MULTISAMPLE_QUALITY_LEVELS_FLAG_NONE;
msQualityLevels.NumQualityLevels = 0;
ThrowIfFailed(d3dDevice_->CheckFeatureSupport(
    D3D12_FEATURE_MULTISAMPLE_QUALITY_LEVELS,
    &msQualityLevels,
    sizeof(msQualityLevels)));

msaaQuality_ = msQualityLevels.NumQualityLevels;
assert(m4xMsaaQuality > 0 && "Unexpected MSAA quality level.");
```

### 创建命令队列和命令列表

**创建命令队列**

```cc
typedef struct D3D12_COMMAND_QUEUE_DESC {
  D3D12_COMMAND_LIST_TYPE   	Type;		// 
  INT                       	Priority;
  D3D12_COMMAND_QUEUE_FLAGS 	Flags;
  UINT                      	NodeMask;
} D3D12_COMMAND_QUEUE_DESC;

HRESULT CreateCommandQueue(
  const D3D12_COMMAND_QUEUE_DESC  *pDesc,			// 描述结构体
  REFIID                           riid,			// COM ID
  void                           **ppCommandQueue	// 接口返回值
);
```

**创建命令分配器**

```cc
/*
 * @type:	命令列表类型: 
 * 			- D3D12_COMMAND_LIST_TYPE_DIRECT: GPU 直接执行的命令
 *			- D3D12_COMMAND_LIST_TYPE_BUNDLE: 命令打包(方便预处理优化命令, 一般不使用)
 * @riid:				COM ID
 * @ppCommandAllcator: 	返回值
 */
HRESULT ID3D12Derived::CreateCommandAllocator(
	D3D12_COMMAND_LIST_TYPE type,
    REFILD riid,
    void **ppCommandAllcator
)
```

**创建命令列表**

```cc
/*
 * @nodeMask:			只有1个GPU时,设置为0, 否则设置物理GPU(不详)
 * @type:				类型和创建分配器时的相同
 *						- D3D12_COMMAND_LIST_TYPE_DIRECT
 *						- D3D12_COMMAND_LIST_TYPE_BUNDLE	
 * @pCommandAllocator:	ID3D12Derived::CreateCommandAllocator 创建的分配器, 
 *						必须与上面的类型一样
 * @pInitialState:		命令列表渲染流水线的初始状态, 使用打包时可传递 nullptr
 * @riid:				COM ID
 * @ppCommandList:		返回值
 */
HRESULT ID3D12Derived::CreateCommandList(
	UINT nodeMask,
    D3D12_COMMAND_LIST_TYPE type,
    ID3D12CommandAllocator *pCommandAllocator,
    I3D312PipelineState *pInitialState,
    REFILD riid,
    void **ppCommandList
)
```

**调用示例**

```cc
ComPtr<ID3D12CommandQueue> 			commandQueue_;
ComPtr<ID3D12CommandAllocator> 		directCmdListAlloc_;
ComPtr<ID3D12GraphicsCommandList> 	commandList_;

void D3DApp::CreateCommandObjects() {
	D3D12_COMMAND_QUEUE_DESC queueDesc = {};
	queueDesc.Type = D3D12_COMMAND_LIST_TYPE_DIRECT;
	queueDesc.Flags = D3D12_COMMAND_QUEUE_FLAG_NONE;
    // 创建命令队列
	ThrowIfFailed(md3dDevice->CreateCommandQueue(
        &queueDesc, IID_PPV_ARGS(&mCommandQueue)
    ));

    // 创建分配器
	ThrowIfFailed(md3dDevice->CreateCommandAllocator(
		D3D12_COMMAND_LIST_TYPE_DIRECT,
		IID_PPV_ARGS(mDirectCmdListAlloc.GetAddressOf())
    ));
	
    // 创建命令列表
	ThrowIfFailed(md3dDevice->CreateCommandList(
		0,
		D3D12_COMMAND_LIST_TYPE_DIRECT,
		mDirectCmdListAlloc.Get(), // Associated command allocator
		nullptr,                   // Initial PipelineStateObject
		IID_PPV_ARGS(mCommandList.GetAddressOf())
    ));
	
    // 创建时需要关闭
	mCommandList->Close();
}
```

### 创建交换链

**交换链结构体**

```cc
typedef struct DXGI_RATIONAL {
  UINT Numerator;		// 分子
  UINT Denominator;		// 分母
} DXGI_RATIONAL;
// 如果设置为 60HZ. Numerator = 60; Denominator = 1;

typedef struct DXGI_MODE_DESC {
    UINT						Width;		// 缓冲区宽		
    UINT						Height;		// 缓冲区高
    DXGI_RATIONAL				RefreshRate;// 刷新率
    DXGI_FORMAT					Format;		// 缓冲区纹理格式
    DXGI_MODE_SCANLINE_ORDER	ScanlineOrdering;// 逐行扫描;隔行扫描
    DXGI_MODE_SCALING			Scanling; // 图形如何相对于屏幕进行拉伸
} DXGI_MODE_DESC;

typedef struct DXGI_SWAP_CHAIN_DESC {
  DXGI_MODE_DESC   	BufferDesc;	// 后台缓冲区的属性.我们仅关注,宽高像素格式
  DXGI_SAMPLE_DESC 	SampleDesc;	// 4XMSAA采样质量
  DXGI_USAGE       	BufferUsage; // DXGI_USAGE_RENDER_TARGET_OUTPUT(表示渲染到后台)
  UINT             	BufferCount; // 缓冲区数量(通常为2双缓冲)
  HWND             	OutputWindow; // 渲染窗口的句柄
  BOOL             	Windowed; // true 窗口模式运行. false 全屏模式运行
  DXGI_SWAP_EFFECT 	SwapEffect;	// 指定为 DXGI_SWAP_EFFECT_FLIP_DISCARD
  UINT             	Flags; // DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH(可选),全屏自动调整
} DXGI_SWAP_CHAIN_DESC;
```

**创建方法**

```cc
HERESULT IDXGIFactory::CreateSwapChain(
	IUnknown              *pDevice,		// 设备
	DXGI_SWAP_CHAIN_DESC  *pDesc,		// 描述结构体
	IDXGISwapChain       **ppSwapChain	// 返回值
);
```

**创建实例**

```cc
DXGI_FORMAT backBufferFormat_ = DXGI_FORMAT_R8G8B8A8_UNORM;
ComPtr<IDXGISwapChain> swapChain_;
constexpr int kSwapChainCount_ = 2;
HWND mainWindow_;

void CreateSwapChain() {
    // 释放旧资源
	swapChain_.Reset();	
    
    // 创建描述符
    DXGI_SWAP_CHAIN_DESC sd;
    sd.BufferDesc.Width = clientWidth;
    sd.BufferDesc.Height = clientHeight;
    sd.BufferDesc.Denominator = 1;
    sd.BufferDesc.Numerator = 60;
    sd.BufferDesc.Format = backBufferFormat_;
    sd.BufferDesc.ScanlineOrdering = DXGI_MODE_SCANLINE_ORDER_UNSPECIFIED;
    sd.BufferDesc.Scaline = DXGI_MODE_SCALING_UNSPECIFIED;
    sd.SampleDesc.Count = msaaState ? : 4 : 1;
    sd.SampleDesc.Quality = msaaState ? (msaaQualityLevel - 1) : 0;
    sd.BufferUsage = DXGI_USAGE_RENDER_TARGET_OUTPUT;
    sd.BufferCount = kSwapChainCount_;
    sd.OutputWindow = mainWindow_;
    sd.Windowed = true;
    sd.SwapEffect = DXGI_SWAP_EFFECT_FLIP_DISCARD;
    sd.Flags = DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH;
    
    // 创建交换链
    ThrowIfFailed(dxgiFactory_->CreateSwapChain(
    	d3dDevice_.Get(),
        &sd,
        &swapChain_
    ));
}
```

### 创建描述符堆

**描述符堆结构体**

```cc
typedef struct D3D12_DESCRIPTOR_HEAP_DESC {
  D3D12_DESCRIPTOR_HEAP_TYPE  	Type;			// 堆类型
  UINT                        	NumDescriptors;	// 描述符数量
  D3D12_DESCRIPTOR_HEAP_FLAGS 	Flags;			// 标志位
  UINT                        	NodeMask;		// 单个适配器取0,否则适配器的Node值
} D3D12_DESCRIPTOR_HEAP_DESC;
```

**调用示例**

```cc
ComPtr<ID3D12DescriptorHeap> rtvHeap_;
ComPtr<ID3D12DescriptorHeap> dsvHeap_;
void CreateRtvAndDsvDescriptorHeaps() {
    D3D12_DESCRIPTOR_HEAP_DESC rtvHeapDesc;
    rtvHeapDesc.NumDescriptor = kSwapChainCount_;
    rtvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_RTV;
    rtvHeapDesc.Flags = D3D12_DESCRITPOR_FLAG_NONE;
    rtvHeapDesc.NodeMask = 0;
    
    ThrowIfFailed(d3dDevice_->CreateDescriptorHeap(
    	&rtvHeapDesc,
        IID_PPV_ARGS(&rtvHeap_)
    ));
    
    D3D12_DESCRIPTOR_HEAP_DESC dsvHeapDesc;
    dsvHeapDesc.NumDescriptor = 1;
    dsvHeapDesc.Type = D3D12_DESCRIPTOR_HEAP_TYPE_DSV;
    dsvHeapDesc.Flags = D3D12_DESCRITPOR_FLAG_NONE;
    dsvHeapDesc.NodeMask = 0;
    
    ThrowIfFailed(d3dDevice_->CreateDescriptorHeap(
    	&dsvHeapDesc,
        IID_PPV_ARGS(&dsvHeap_)
    ));
}
```

## 窗口更改初始化(OnResize) 函数

### 释放旧资源

1. 刷新命令队列, 调用 `ID3D12GraphicsCommandList::Reset`
2. 对缓冲区调用 `Reset` 
3. 修改 `SwapChain` 缓冲区的大小
4. 重置后台缓冲区索引
5. 重新获得 RTV 缓冲区视图
6. 创建新的 深度模板缓冲 
7. 设置视口
8. 设置裁剪矩形

**完整代码如下**

```cc
ComPtr<ID3D12Resource> swapChainBuffer_[kSwapChainBufferCount_];
ComPtr<ID3D12Resource> depthStencilBuffer_;
UINT currBackBuffer_;

void onResize() {
    // setp1 刷新命令队列, 调用 `ID3D12GraphicsCommandList::Reset`
    flushCommandQueue();
    ThrwoIfFailed(commandList_->Reset(commandAlloc_.Get(), nullptr));
	
    // step2 释放缓冲区
    for (int i = 0; i < kSwapChainBufferCount_; ++i)
        swapChainBuffer_[i].Reset();
    depthStencilBuffer_.Reset();
    
    // step3 修改 SwapChain 缓冲区的大小
    ThrowIfFailed(swapChain->ResizeBuffers(
    	kSwapChainBufferCount,
        mClientWidth, mClientHeight,
        backBufferFormat_,
        DXGI_SWAP_CHAIN_FLAG_ALLOW_MODE_SWITCH
    ));
    
    // step4 重置后台缓冲区索引
    currBackBuffer_ = 0;
    
    // step5 获得 RTV 缓冲区视图
    CD3DX12_CPU_DESCRIPTOR_HANDLE rtvHeapHandle(
        rtvHeap_->GetCPUDescritproHandleForHeapStart()
    );
    for (int i = 0; i < kSwapChainBufferCount; ++i) {
        ThrowIfFailed(swapChain_->GetBuffer(
            i, 
            IID_PPV_ARGS(&swapChainBuffer_[i])
        ));
        d3dDevice_->CreateRenderTargetView(
            swapChainBuffer_[i].Get(), nullptr, rtvHeapHandle
        );
        rtvHeapHandle.Offset(1, rtvDescriptorSize);
    }
    
    // step6 深度模板缓冲 
    D3D12_RESOURCE_DESC depthStencilDesc;
	depthStencilDesc.Dimension = D3D12_RESOURCE_DIMENSION_TEXTURE2D;
    depthStencilDesc.Alignment = 0;
    depthStencilDesc.Width = mClientWidth;
    depthStencilDesc.Height = mClientHeight;
    depthStencilDesc.DepthOrArraySize = 1;
    depthStencilDesc.MipLevels = 1;
    depthStencilDesc.Format = DXGI_FORMAT_R24G8_TYPELESS;
    depthStencilDesc.SampleDesc.Count = m4xMsaaState ? 4 : 1;
    depthStencilDesc.SampleDesc.Quality = m4xMsaaState ? (m4xMsaaQuality - 1) : 0;
    depthStencilDesc.Layout = D3D12_TEXTURE_LAYOUT_UNKNOWN;
    depthStencilDesc.Flags = D3D12_RESOURCE_FLAG_ALLOW_DEPTH_STENCIL;
    
    D3D12_CLEAR_VALUE optClear;
    optClear.Format = mDepthStencilFormat;
    optClear.DepthStencil.Depth = 1.0f;
    optClear.DepthStencil.Stencil = 0;
    ThrowIfFailed(md3dDevice->CreateCommittedResource(
        &CD3DX12_HEAP_PROPERTIES(D3D12_HEAP_TYPE_DEFAULT),
		D3D12_HEAP_FLAG_NONE,
        &depthStencilDesc,
		D3D12_RESOURCE_STATE_COMMON,
        &optClear,
        IID_PPV_ARGS(mDepthStencilBuffer.GetAddressOf())
    ));
    
    // 为 深度模板 mipmap 0 缓冲区设置创建视图
    D3D12_DEPTH_STENCIL_VIEW_DESC dsvDesc;
	dsvDesc.Flags = D3D12_DSV_FLAG_NONE;
	dsvDesc.ViewDimension = D3D12_DSV_DIMENSION_TEXTURE2D;
	dsvDesc.Format = mDepthStencilFormat;
	dsvDesc.Texture2D.MipSlice = 0;
    md3dDevice->CreateDepthStencilView(
        depthStencilBuffer_.Get(), &dsvDesc, DepthStencilView()
    );
    
    // 修改深度模板缓冲区资源的状态为写
    mCommandList->ResourceBarrier(
        1, 
        &CD3DX12_RESOURCE_BARRIER::Transition(
            mDepthStencilBuffer.Get(),
			D3D12_RESOURCE_STATE_COMMON, 
            D3D12_RESOURCE_STATE_DEPTH_WRITE
        )
    );
    
    // 执行 resize 命令
	ThrowIfFailed(mCommandList->Close());
    ID3D12CommandList* cmdsLists[] = { mCommandList.Get() };
    mCommandQueue->ExecuteCommandLists(_countof(cmdsLists), cmdsLists);
	FlushCommandQueue();
    
    // step7 设置视口
    mScreenViewport.TopLeftX = 0;
	mScreenViewport.TopLeftY = 0;
	mScreenViewport.Width    = static_cast<float>(mClientWidth);
	mScreenViewport.Height   = static_cast<float>(mClientHeight);
	mScreenViewport.MinDepth = 0.0f;
	mScreenViewport.MaxDepth = 1.0f;
	
    // step8 设置裁剪矩形
	mScissorRect = { 0, 0, mClientWidth, mClientHeight };
}
```

## 渲染主循环

1. 释放命令列表分配器
2. 释放命令列表
3. 修改 RTV 的状态为 `RENDER_TARGET`
4. 设置视口, 设置裁剪矩形
5. 清除 RTV, DSV
6. 绑定 RTV, DSV 到流水线
7. 绘制几何
8. 切换 RTV 的状态为 `PRESENT`
9. `close commandList` 命令队列执行 `ExecuteCommandLists`
10. 交换缓冲区
11. 刷新任务队列

```cc
void update() {
    // step1
    ThrowIfFailed(directCmdListAlloc_->Reset());	
    // step2
    ThrowIfFailed(commandList_->Reset(directCmdListAlloc_.Get(), nullptr));
    // stpe3
    commandList_->Resourcebarrier(1, 
		&CD3DX12_RESOURCE_BARRIER::Transition(
        	CurrentBackBuffer(), 
            D3D12_RESOURCE_STATE_PRESENT, 
            D3D12_RESOURCE_STATE_RENDER_TARGET
    	));
    // step4
    commandList_->RSSetViewports(1, &screenViewport_);
    commandList_->RSSetScissorRects(1, &scissiorRect_);
	// step5
	pCommandList_->ClearRenderTargetView(currentBackBufferView(), 			
	DX::Colors::LightBlue, 0, nullptr);
	pCommandList_->ClearDepthStencilView(depthStencilBufferView(),
		D3D12_CLEAR_FLAG_DEPTH | D3D12_CLEAR_FLAG_STENCIL, 1.0f, 0, 0, nullptr
	);
    // step6
    commandList_->OMSetRenderTarget(
        1, &CurrentBackBufferView(), 
        true, &DepthStencilView()
    );
    // step7 draw...
	ThrowIfFailed(commandList_->Close());
    ID3D12CommandList *cmdsLists[] = { commandList_.Get() };
    commandQueue_->ExecuteCommandLists(size(cmdsLists), cmdsLists);
    // step8
    ThrowIfFailed(swapChain_->Present(0, 0));
    currentBackBuffer_ = (currentBackBuffer_ + 1) % kSwapChainBufferCount_;
    
    // 刷新任务队列
    flushCommandQueue();
}
```











