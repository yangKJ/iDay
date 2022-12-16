---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第3天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

**本文正在参加[「金石计划 . 瓜分6万现金大奖」](https://juejin.cn/post/7162096952883019783 " https://juejin.cn/post/7162096952883019783 ")**

本案例的目的是理解如何用Metal实现3x3卷积矩阵效果滤镜，取像素点周边九个区域半径点像素rgb值进行矩阵运算获取新的rgb值;

目前有如下几款卷积核供使用，

- default：空卷积核
- identity：原始卷积核
- edgedetect：边缘检测卷积核
- embossment：浮雕滤波器卷积核
- embossment45：45度的浮雕滤波器卷积核
- morphological：侵蚀卷积核
- laplance：拉普拉斯算子，边缘检测算子
- sharpen：锐化卷积核
- sobel：边缘提取卷积核，求梯度比较常用

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 锐化卷积效果滤镜
let filter = C7ConvolutionMatrix3x3(convolutionType: .sharpen(iterations: 2))

// 方案1:
let dest = BoxxIO.init(element: originImage, filter: filter)
ImageView.image = try? dest.output()

dest.filters.forEach {
    NSLog("%@", "\($0.parameterDescription)")
}

// 方案2:
ImageView.image = try? originImage.make(filter: filter)

// 方案3:
ImageView.image = originImage ->> filter
```

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7ConvolutionMatrix3x3")`，参数因子`[Float(convolutionPixel)]`

对外开放参数
- `convolutionPixel`: 卷积像素

```
/// 3 x 3卷积
public struct C7ConvolutionMatrix3x3: C7FilterProtocol {
    
    public enum ConvolutionType {
        case `default`
        case identity
        case edgedetect
        case embossment
        case embossment45
        case morphological
        case sobel(orientation: Bool)
        case laplance(iterations: Float)
        case sharpen(iterations: Float)
        case custom(Matrix3x3)
    }
    
    /// Convolution pixels, default 1
    public var convolutionPixel: Int = 1
    private var matrix: Matrix3x3
    
    public var modifier: Modifier {
        return .compute(kernel: "C7ConvolutionMatrix3x3")
    }
    
    public var factors: [Float] {
        return [Float(convolutionPixel)]
    }
    
    public func setupSpecialFactors(for encoder: MTLCommandEncoder, index: Int) {
        guard let computeEncoder = encoder as? MTLComputeCommandEncoder else { return }
        var factor = matrix.to_factor()
        computeEncoder.setBytes(&factor, length: Matrix3x3.size, index: index + 1)
    }
    
    public init(matrix: Matrix3x3) {
        self.matrix = matrix
    }
    
    public init(convolutionType: ConvolutionType) {
        self.init(matrix: convolutionType.matrix)
    }
    
    public mutating func updateConvolutionType(_ convolutionType: ConvolutionType) {
        self.matrix = convolutionType.matrix
    }
    
    public mutating func updateMatrix3x3(_ matrix: Matrix3x3) {
        self.matrix = matrix
    }
}

extension C7ConvolutionMatrix3x3.ConvolutionType {
    var matrix: Matrix3x3 {
        switch self {
        case .identity:
            return Matrix3x3.Kernel.identity
        case .edgedetect:
            return Matrix3x3.Kernel.edgedetect
        case .embossment:
            return Matrix3x3.Kernel.embossment
        case .embossment45:
            return Matrix3x3.Kernel.embossment45
        case .morphological:
            return Matrix3x3.Kernel.morphological
        case .sobel(let orientation):
            return Matrix3x3.Kernel.sobel(orientation)
        case .laplance(let iterations):
            return Matrix3x3.Kernel.laplance(iterations)
        case .sharpen(let iterations):
            return Matrix3x3.Kernel.sharpen(iterations)
        case .custom(let matrix3x3):
            return matrix3x3
        default:
            return Matrix3x3.Kernel.`default`
        }
    }
}
```

- 着色器

取像素点周边九个区域半径点像素，然后归一化处理，然后取出每个像素对应rgb，再进行卷积矩阵运算得到卷积之后的rgb值，生成新的像素颜色；

```
kernel void C7ConvolutionMatrix3x3(texture2d<half, access::write> outputTexture [[texture(0)]],
                                   texture2d<half, access::sample> inputTexture [[texture(1)]],
                                   constant float *pixel [[buffer(0)]],
                                   constant float3x3 *matrix3x3 [[buffer(1)]],
                                   uint2 grid [[thread_position_in_grid]]) {
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const float x = float(grid.x);
    const float y = float(grid.y);
    const float w = float(inputTexture.get_width());
    const float h = float(inputTexture.get_height());
    const float l = float(x - *pixel);
    const float r = float(x + *pixel);
    const float t = float(y - *pixel);
    const float b = float(y + *pixel);
    
    // Normalization
    const float2 m11Coordinate = float2(l / w, t / h);
    const float2 m12Coordinate = float2(x / w, t / h);
    const float2 m13Coordinate = float2(r / w, t / h);
    const float2 m21Coordinate = float2(l / w, y / h);
    const float2 m22Coordinate = float2(x / w, y / h);
    const float2 m23Coordinate = float2(r / w, y / h);
    const float2 m31Coordinate = float2(l / w, b / h);
    const float2 m32Coordinate = float2(x / w, b / h);
    const float2 m33Coordinate = float2(r / w, b / h);
    
    const half4 centerColor = inputTexture.sample(quadSampler, m22Coordinate);
    
    const half3 m11Color = inputTexture.sample(quadSampler, m11Coordinate).rgb;
    const half3 m12Color = inputTexture.sample(quadSampler, m12Coordinate).rgb;
    const half3 m13Color = inputTexture.sample(quadSampler, m13Coordinate).rgb;
    const half3 m21Color = inputTexture.sample(quadSampler, m21Coordinate).rgb;
    const half3 m22Color = centerColor.rgb;
    const half3 m23Color = inputTexture.sample(quadSampler, m23Coordinate).rgb;
    const half3 m31Color = inputTexture.sample(quadSampler, m31Coordinate).rgb;
    const half3 m32Color = inputTexture.sample(quadSampler, m32Coordinate).rgb;
    const half3 m33Color = inputTexture.sample(quadSampler, m33Coordinate).rgb;
    
    const float3x3 matrix = (*matrix3x3);
    half3 resultColor = half3(0.0h);
    resultColor += m11Color * (matrix[0][0]) + m12Color * (matrix[0][1]) + m13Color * (matrix[0][2]);
    resultColor += m21Color * (matrix[1][0]) + m22Color * (matrix[1][1]) + m23Color * (matrix[1][2]);
    resultColor += m31Color * (matrix[2][0]) + m32Color * (matrix[2][1]) + m33Color * (matrix[2][2]);
    
    const half4 outColor = half4(resultColor, centerColor.a);
    outputTexture.write(outColor, grid);
}
```

### 其他卷积核

```
extension Matrix3x3 {
    /// 常见 3x3 矩阵卷积内核，考线性代数时刻😪
    /// Common 3x3 matrix convolution kernel
    public struct Kernel { }
}

extension Matrix3x3.Kernel {
    /// 原始矩阵，空卷积核
    /// The original matrix, the empty convolution kernel
    public static let `default` = Matrix3x3(values: [
        0.0, 0.0, 0.0,
        0.0, 1.0, 0.0,
        0.0, 0.0, 0.0,
    ])

    public static let identity = Matrix3x3(values: [
        1.0, 0.0, 0.0,
        0.0, 1.0, 0.0,
        0.0, 0.0, 1.0,
    ])

    /// 边缘检测矩阵
    /// Edge detection matrix
    public static let edgedetect = Matrix3x3(values: [
        -1.0, -1.0, -1.0,
        -1.0,  8.0, -1.0,
        -1.0, -1.0, -1.0,
    ])

    /// 浮雕矩阵
    /// Anaglyph matrix
    public static let embossment = Matrix3x3(values: [
        -2.0, 0.0, 0.0,
         0.0, 1.0, 0.0,
         0.0, 0.0, 2.0,
    ])

    /// 45度的浮雕滤波器
    /// A 45 degree emboss filter
    public static let embossment45 = Matrix3x3(values: [
        -1.0, -1.0, 0.0,
        -1.0,  0.0, 1.0,
         0.0,  1.0, 1.0,
    ])

    /// 侵蚀矩阵
    /// Matrix erosion
    public static let morphological = Matrix3x3(values: [
        1.0, 1.0, 1.0,
        1.0, 1.0, 1.0,
        1.0, 1.0, 1.0,
    ])
    
    /// 拉普拉斯算子，边缘检测算子
    /// Laplace operator, edge detection operator
    public static func laplance(_ iterations: Float) -> Matrix3x3 {
        let xxx = iterations
        return Matrix3x3(values: [
             0.0, -1.0,  0.0,
            -1.0,  xxx, -1.0,
             0.0, -1.0,  0.0,
        ])
    }
    
    /// 锐化矩阵
    /// Sharpening matrix
    public static func sharpen(_ iterations: Float) -> Matrix3x3 {
        let cc = (8 * iterations + 1)
        let xx = (-iterations)
        return Matrix3x3(values: [
            xx, xx, xx,
            xx, cc, xx,
            xx, xx, xx,
        ])
    }
    
    /// Sobel矩阵图像边缘提取，求梯度比较常用
    /// Sobel matrix image edge extraction, gradient is more commonly used
    public static func sobel(_ orientation: Bool) -> Matrix3x3 {
        if orientation {
            return Matrix3x3(values: [
                -1.0, 0.0, 1.0,
                -2.0, 0.0, 2.0,
                -1.0, 0.0, 1.0,
            ])
        } else {
            return Matrix3x3(values: [
                -1.0, -2.0, -1.0,
                 0.0,  0.0,  0.0,
                 1.0,  2.0,  1.0,
            ])
        }
    }
    
    /// BT.601, which is the standard for SDTV.
    public static let to601 = Matrix3x3(values: [
        1.164,  1.164, 1.164,
        0.000, -0.392, 2.017,
        1.596, -0.813, 0.000,
    ])
    
    /// BT.601 full range (ref: http://www.equasys.de/colorconversion.html)
    public static let to601FullRange = Matrix3x3(values: [
        1.0,  1.000, 1.000,
        0.0, -0.343, 1.765,
        1.4, -0.711, 0.000,
    ])
    
    /// BT.709, which is the standard for HDTV.
    public static let to709 = Matrix3x3(values: [
        1.164,  1.164, 1.164,
        0.000, -0.213, 2.112,
        1.793, -0.533, 0.000,
    ])
}
```

### 效果图

- 常见核卷积图

| 边缘检测矩阵 | 浮雕矩阵 | 45度的浮雕滤波器 |
|:-:|:-:|:-:|
|![WX20221125-163024.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7a1b31e1eb7c4be3b15d2052759ef192~tplv-k3u1fbpfcp-watermark.image?)|![WX20221125-163116.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/73b4cfa5280a4ca5b33a64c328477018~tplv-k3u1fbpfcp-watermark.image?)|![WX20221125-163210.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1b66393f7e6a41648822693c971e1646~tplv-k3u1fbpfcp-watermark.image?)|
| 锐化矩阵 | 拉普拉斯算子 | Sobel矩阵图像边缘提取 |
|![WX20221121-144331.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e2e07943fd9d43188f0457a395ef837f~tplv-k3u1fbpfcp-watermark.image?)|![WX20221125-163703.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1dcbb80444cb4732869b50db18bf2d34~tplv-k3u1fbpfcp-watermark.image?)|![WX20221125-163826.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ea5c3044c5d64c1190007f876c2e820f~tplv-k3u1fbpfcp-watermark.image?)|

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

- 关于3x3矩阵卷积效果滤镜介绍与设计到此为止吧。
- 慢慢再补充其他相关滤镜，喜欢就给我点个星🌟吧。
- [**滤镜Demo地址**](https://github.com/yangKJ/Harbeth)，目前包含`100+`种滤镜，同时也支持CoreImage混合使用。
- 再附上一个开发加速库[**KJCategoriesDemo地址**](https://github.com/yangKJ/KJCategories)
- 再附上一个网络基础库[**RxNetworksDemo地址**](https://github.com/yangKJ/RxNetworks)
- 喜欢的老板们可以点个星🌟，谢谢各位老板！！！
  
✌️.
