---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第12天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现图像波动效果滤镜，还可类似涂鸦效果，主要就是对纹理坐标进行正余弦偏移处理；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 波动效果
let filter = C7Fluctuate.init(extent: 50, amplitude: 0.003, fluctuate: 2.5)

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

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Fluctuate")`，参数因子`[extent, amplitude, fluctuate]`

对外开放参数
- `amplitude`: 控制振幅的大小，越大图像越夸张；
- `extent`: 控制波动程度，越大越密集；
- `fluctuate`: 波动幅度；

```
/// 波动效果，还可类似涂鸦效果
public struct C7Fluctuate: C7FilterProtocol {
    
    /// 控制振幅的大小，越大图像越夸张
    /// Control the size of the amplitude, the larger the image, the more exaggerated the image.
    public var amplitude: Float = 0.002
    public var extent: Float = 50.0
    public var fluctuate: Float = 0.5
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Fluctuate")
    }
    
    public var factors: [Float] {
        return [extent, amplitude, fluctuate]
    }
    
    public init(extent: Float = 50.0, amplitude: Float = 0.002, fluctuate: Float = 0.5) {
        self.extent = extent
        self.amplitude = amplitude
        self.fluctuate = fluctuate
    }
}
```

- 着色器

纹理坐标归一化处理，然后获取到偏移正余弦值作为xy，取出纹理像素颜色；  

```
kernel void C7Fluctuate(texture2d<half, access::write> outputTexture [[texture(0)]],
                        texture2d<half, access::sample> inputTexture [[texture(1)]],
                        device float *extent [[buffer(0)]],
                        device float *amplitude [[buffer(1)]],
                        device float *fluctuate [[buffer(2)]],
                        uint2 grid [[thread_position_in_grid]]) {
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const float2 textureCoordinate = float2(float(grid.x) / outputTexture.get_width(), float(grid.y) / outputTexture.get_height());
    
    float2 offset = float2(0, 0);
    offset.x = sin(grid.x * *extent + *fluctuate) * *amplitude;
    offset.y = cos(grid.y * *extent + *fluctuate) * *amplitude;
    
    const float2 tx = textureCoordinate + offset;//mix(textureCoordinate, textureCoordinate+offset, 0.01);
    const half4 outColor = inputTexture.sample(quadSampler, tx);
    
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
