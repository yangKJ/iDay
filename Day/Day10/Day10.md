---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第10天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现图像阀值素描滤镜，用于图像阀值素描，形成有噪点的素描；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 用于图像阀值素描，形成有噪点的素描
let filter = C7ThresholdSketch.init(edgeStrength: 2.5, threshold: 0.25)

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

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7ThresholdSketch")`，参数因子`[edgeStrength, threshold]`

对外开放参数
- `threshold`: 任何高于这个阈值的边缘都是黑色的，低于白色的任何东西都是黑色的。
- `edgeStrength`: 调整滤波器的动态范围。更高的值会导致更强的边缘，但可以饱和强度的色彩空间。

```
public struct C7ThresholdSketch: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: 0.0, max: 1.0, value: 0.25)
    
    /// Any edge above this threshold will be black, and anything below white. Ranges from 0.0 to 1.0
    @ZeroOneRange public var threshold: Float = range.value

    /// Adjusts the dynamic range of the filter.
    /// Higher values lead to stronger edges, but can saturate the intensity colorspace.
    public var edgeStrength: Float = 1
    
    public var modifier: Modifier {
        return .compute(kernel: "C7ThresholdSketch")
    }
    
    public var factors: [Float] {
        return [edgeStrength, threshold]
    }
    
    public init(edgeStrength: Float = 1, threshold: Float = range.value) {
        self.edgeStrength = edgeStrength
        self.threshold = threshold
    }
}
```

- 着色器

取出周边1像素对应点的归一化坐标，获取到这些点对应的红色值；  
水平方向取顶部和底部红色值做个整合处理，竖直方向取左边和右边做个整合处理，分别得到(h, v);  
`length`计算出范围值，`step`计算出阈值对应的颜色，最后得到黑色或白色像素颜色即可；

```
kernel void C7ThresholdSketch(texture2d<half, access::write> outputTexture [[texture(0)]],
                              texture2d<half, access::sample> inputTexture [[texture(1)]],
                              constant float *edgeStrength [[buffer(0)]],
                              constant float *threshold [[buffer(1)]],
                              uint2 grid [[thread_position_in_grid]]) {
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    
    const float x = float(grid.x);
    const float y = float(grid.y);
    const float width = float(inputTexture.get_width());
    const float height = float(inputTexture.get_height());
    
    const float2 leftCoordinate = float2((x - 1) / width, y / height);
    const float2 rightCoordinate = float2((x + 1) / width, y / height);
    const float2 topCoordinate = float2(x / width, (y - 1) / height);
    const float2 bottomCoordinate = float2(x / width, (y + 1) / height);
    const float2 topLeftCoordinate = float2((x - 1) / width, (y - 1) / height);
    const float2 topRightCoordinate = float2((x + 1) / width, (y - 1) / height);
    const float2 bottomLeftCoordinate = float2((x - 1) / width, (y + 1) / height);
    const float2 bottomRightCoordinate = float2((x + 1) / width, (y + 1) / height);
    
    const half leftIntensity = inputTexture.sample(quadSampler, leftCoordinate).r;
    const half rightIntensity = inputTexture.sample(quadSampler, rightCoordinate).r;
    const half topIntensity = inputTexture.sample(quadSampler, topCoordinate).r;
    const half bottomIntensity = inputTexture.sample(quadSampler, bottomCoordinate).r;
    const half topLeftIntensity = inputTexture.sample(quadSampler, topLeftCoordinate).r;
    const half topRightIntensity = inputTexture.sample(quadSampler, topRightCoordinate).r;
    const half bottomLeftIntensity = inputTexture.sample(quadSampler, bottomLeftCoordinate).r;
    const half bottomRightIntensity = inputTexture.sample(quadSampler, bottomRightCoordinate).r;
    
    half h = -topLeftIntensity - 2.0h * topIntensity - topRightIntensity + bottomLeftIntensity + 2.0h * bottomIntensity + bottomRightIntensity;
    h = max(0.0h, h);
    half v = -bottomLeftIntensity - 2.0h * leftIntensity - topLeftIntensity + bottomRightIntensity + 2.0h * rightIntensity + topRightIntensity;
    v = max(0.0h, v);
    
    half mag = length(half2(h, v)) * half(*edgeStrength);
    mag = 1.0h - step(half(*threshold), mag);
    
    const half4 outColor = half4(half3(mag), 1.0h);
    outputTexture.write(outColor, grid);
}
```

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
