---
theme: cyanosis
---
开启掘金成长之旅！这是我参与「掘金日新计划 · 12 月更文挑战」的第27天，[点击查看活动详情](https://juejin.cn/post/7167294154827890702)

本案例的目的是理解如何用Metal实现色彩空间转换效果滤镜，转换在不同色彩空间生成的图像；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 色彩空间转换滤镜
let filter = C7ColorSpace.init(with: .rgb_to_yuv)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下效果

|rgb_to_yiq|yiq_to_rgb|rgb_to_yuv|
|:-:|:-:|:-:|
|![WX20221220-144832.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cc90d79761764cb4b4a3e018b85bdf6b~tplv-k3u1fbpfcp-watermark.image?)|![WX20221220-135844.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b930e14fc32d47f6bcae4e7489f2a8c0~tplv-k3u1fbpfcp-watermark.image?)|![WX20221220-143759.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/788119c9432a4af9a6dc3eef20846937~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: type.rawValue)`；

```
/// 色彩空间转换
public struct C7ColorSpace: C7FilterProtocol {
    
    public enum SwapType: String, CaseIterable {
        case rgb_to_yiq = "C7ColorSpaceRGB2YIQ"
        case yiq_to_rgb = "C7ColorSpaceYIQ2RGB"
        case rgb_to_yuv = "C7ColorSpaceRGB2YUV"
        case yuv_to_rgb = "C7ColorSpaceYUV2RGB"
    }
    
    private let type: SwapType
    
    public var modifier: Modifier {
        return .compute(kernel: type.rawValue)
    }
    
    public init(with type: SwapType) {
        self.type = type
    }
}
```

- 着色器

每条通道乘以各自偏移求和得到Y，用Y作为新的像素rgb； 

```
kernel void C7ColorSpaceRGB2Y(texture2d<half, access::write> outputTexture [[texture(0)]],
                              texture2d<half, access::read> inputTexture [[texture(1)]],
                              uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half Y = half((0.299 * inColor.r) + (0.587 * inColor.g) + (0.114 * inColor.b));
    const half4 outColor = half4(Y, Y, Y, inColor.a);
    
    outputTexture.write(outColor, grid);
}

// See: https://en.wikipedia.org/wiki/YIQ
kernel void C7ColorSpaceRGB2YIQ(texture2d<half, access::write> outputTexture [[texture(0)]],
                                texture2d<half, access::read> inputTexture [[texture(1)]],
                                uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half3x3 RGBtoYIQ = half3x3({0.299, 0.587, 0.114}, {0.596, -0.274, -0.322}, {0.212, -0.523, 0.311});
    const half3 yiq = RGBtoYIQ * inColor.rgb;
    const half4 outColor = half4(yiq, inColor.a);
    
    outputTexture.write(outColor, grid);
}

kernel void C7ColorSpaceYIQ2RGB(texture2d<half, access::write> outputTexture [[texture(0)]],
                                texture2d<half, access::read> inputTexture [[texture(1)]],
                                uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half3x3 YIQtoRGB = half3x3({1.0, 0.956, 0.621}, {1.0, -0.272, -0.647}, {1.0, -1.105, 1.702});
    const half3 rgb = YIQtoRGB * inColor.rgb;
    const half4 outColor = half4(rgb, inColor.a);
    
    outputTexture.write(outColor, grid);
}

// See: https://en.wikipedia.org/wiki/YUV
kernel void C7ColorSpaceRGB2YUV(texture2d<half, access::write> outputTexture [[texture(0)]],
                                texture2d<half, access::read> inputTexture [[texture(1)]],
                                uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half3x3 RGBtoYUV = half3x3({0.299, 0.587, 0.114}, {-0.299, -0.587, 0.886}, {0.701, -0.587, -0.114});
    const half3 yuv = RGBtoYUV * inColor.rgb;
    const half4 outColor = half4(yuv, inColor.a);
    
    outputTexture.write(outColor, grid);
}

kernel void C7ColorSpaceYUV2RGB(texture2d<half, access::write> outputTexture [[texture(0)]],
                                texture2d<half, access::read> inputTexture [[texture(1)]],
                                uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);

    const half3x3 YUVtoRGB = half3x3({1.0, 0.0, 1.28033}, {1.0, -0.21482, -0.38059}, {1.0, 2.21798, 0.0});
    const half3 rgb = YUVtoRGB * inColor.rgb;
    const half4 outColor = half4(rgb, inColor.a);

    outputTexture.write(outColor, grid);
}
```

### 色彩空间

- YIQ

在YIQ系统中，是NTSC（National Television Standards Committee）电视系统标准；

- Y是提供黑白电视及彩色电视的亮度信号Luminance，即亮度Brightness；
- I代表In-phase，色彩从橙色到青色；
- Q代表Quadrature-phase，色彩从紫色到黄绿色；

<p align="left">
<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dff20ee49afa4380bb64199e7e64f74c~tplv-k3u1fbpfcp-watermark.image" width="300" hspace="1px">
</p>

转换公式如下：

<p align="left">
<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/145204d50db4470bb8fe70e830a365b2~tplv-k3u1fbpfcp-watermark.image" width="500" hspace="1px">
</p>

- YUV

YUV是在工程师想要在黑白基础设施中使用彩色电视时发明的。他们需要一种信号传输方法，既能与黑白 (B&W) 电视兼容，又能添加颜色。亮度分量已经作为黑白信号存在；他们将紫外线信号作为解决方案添加到其中。

```
由于 U 和 V 是色差信号，因此在直接 R 和 B 信号上选择色度的 UV 表示。换句话说，U 和 V 信号告诉电视在不改变亮度的情况下改变某个点的颜色。
或者 U 和 V 信号告诉显示器以牺牲另一种颜色为代价使一种颜色更亮，以及它应该移动多少。
U 和 V 值越高（或负值越低），斑点的饱和度（色彩）就越高。
U 值和 V 值越接近零，颜色偏移越小，这意味着红、绿和蓝光的亮度会更均匀，从而产生更灰的点。
这是使用色差信号的好处，即不是告诉颜色有多少红色，而是告诉红色比绿色或蓝色多多少。
反过来，这意味着当 U 和 V 信号为零或不存在时，它只会显示灰度图像。
如果使用 R 和 B，即使在黑白场景中，它们也将具有非零值，需要所有三个数据承载信号。
这在早期的彩色电视中很重要，因为旧的黑白电视信号没有 U 和 V 信号，这意味着彩色电视开箱后只会显示为黑白电视。
此外，黑白接收器可以接收 Y' 信号并忽略 U 和 V 颜色信号，使 YUV 向后兼容所有现有的黑白设备、输入和输出。
如果彩色电视标准不使用色差信号，这可能意味着彩色电视会从 B& 中产生有趣的颜色 W 广播，否则需要额外的电路将黑白信号转换为彩色。
有必要为色度通道分配较窄的带宽，因为没有可用的额外带宽。
如果某些亮度信息是通过色度通道到达的（如果使用 RB 信号而不是差分 UV 信号，就会出现这种情况），黑白分辨率就会受到影响。
```

YUV 模型定义了一个亮度分量 (Y)，表示物理线性空间亮度，以及两个色度分量，分别称为 U（蓝色投影）和 V（红色投影）。它可用于在 RGB 模型之间进行转换，并具有不同的颜色空间

<p align="left">
<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/635111f7e2a64820abcb44efdf24086d~tplv-k3u1fbpfcp-watermark.image" width="300" hspace="1px">
</p>

转换公式如下：

<p align="left">
<img src="https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9176ffad0b03410396626fa42d0fe6b4~tplv-k3u1fbpfcp-watermark.image" width="500" hspace="1px">
</p>

### [Harbeth功能清单](https://github.com/yangKJ/Harbeth)

- 支持ios系统和macOS系统
- 支持运算符函数式操作
- 支持多种模式数据源 UIImage, CIImage, CGImage, CMSampleBuffer, CVPixelBuffer.
- 支持快速设计滤镜
- 支持合并多种滤镜效果
- 支持输出源的快速扩展
- 支持相机采集特效
- 支持视频添加滤镜特效
- 支持矩阵卷积
- 支持使用系统 MetalPerformanceShaders.
- 支持兼容 CoreImage.
- 滤镜部分大致分为以下几个模块：
   - [x] [Blend](https://github.com/yangKJ/Harbeth/tree/master/Sources/Compute/Blend)：图像融合技术
   - [x] [Blur](https://github.com/yangKJ/Harbeth/tree/master/Sources/Compute/Blur)：模糊效果
   - [x] [Pixel](https://github.com/yangKJ/Harbeth/tree/master/Sources/Compute/ColorProcess)：图像的基本像素颜色处理
   - [x] [Effect](https://github.com/yangKJ/Harbeth/tree/master/Sources/Compute/Effect)：效果处理
   - [x] [Lookup](https://github.com/yangKJ/Harbeth/tree/master/Sources/Compute/Lookup)：查找表过滤器
   - [x] [Matrix](https://github.com/yangKJ/Harbeth/tree/master/Sources/Compute/Matrix): 矩阵卷积滤波器
   - [x] [Shape](https://github.com/yangKJ/Harbeth/tree/master/Sources/Compute/Shape)：图像形状大小相关
   - [x] [Visual](https://github.com/yangKJ/Harbeth/tree/master/Sources/Compute/Visual): 视觉动态特效
   - [x] [MPS](https://github.com/yangKJ/Harbeth/tree/master/Sources/Compute/MPS): 系统 MetalPerformanceShaders.

### 最后

- 慢慢再补充其他相关滤镜，喜欢就给我点个星🌟吧。
- [**滤镜Demo地址**](https://github.com/yangKJ/Harbeth)，目前包含`100+`种滤镜，同时也支持CoreImage混合使用。
- 再附上一个开发加速库[**KJCategoriesDemo地址**](https://github.com/yangKJ/KJCategories)
- 再附上一个网络基础库[**RxNetworksDemo地址**](https://github.com/yangKJ/RxNetworks)
- 喜欢的老板们可以点个星🌟，谢谢各位老板！！！

✌️.
