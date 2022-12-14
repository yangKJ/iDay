---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第8天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现调节胶片颗粒感滤镜，通过调整颗粒参数来调整晶粒尺寸来达到颗粒感效果；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)

### 实操代码

```
// 调节胶片颗粒感滤镜
let filter = C7Granularity.init(grain: 0.7)

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

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Granularity")`，参数因子`[grain]`

对外开放参数
- `grain`: 通过调整颗粒参数来调整晶粒尺寸，大小从0.0到0.5不等，其中0.0代表原始图像。

```
/// 调节胶片颗粒感
public struct C7Granularity: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: 0.0, max: 0.5, value: 0.3)
    
    /// The grain size is adjusted by adjusting the grain parameter. The grain size ranges from 0.0 to 0.5,
    /// Where 0.0 represents the original image,
    public var grain: Float = range.value
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Granularity")
    }
    
    public var factors: [Float] {
        return [grain]
    }
    
    public init(grain: Float = range.value) {
        self.grain = grain
    }
}
```

- 着色器

对坐标点归一化处理，计算出小数部分`noise`，最后对每个像素x颗粒度；

```
kernel void C7Granularity(texture2d<half, access::write> outputTexture [[texture(0)]],
                          texture2d<half, access::read> inputTexture [[texture(1)]],
                          constant float *grain [[buffer(0)]],
                          uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    const float2 textureCoordinate = float2(float(grid.x) / outputTexture.get_width(), float(grid.y) / outputTexture.get_height());
    
    const float d = dot(textureCoordinate, float2(12.9898, 78.233) * 2.0);
    const half noise = half(fract(sin(d) * 43758.5453h));
    const half4 outColor = half4(inColor - noise * (*grain));
    
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

- 关于调整胶片颗粒对滤镜介绍与设计到此为止吧。
- 慢慢再补充其他相关滤镜，喜欢就给我点个星🌟吧。
- [**滤镜Demo地址**](https://github.com/yangKJ/Harbeth)，目前包含`100+`种滤镜，同时也支持CoreImage混合使用。
- 再附上一个开发加速库[**KJCategoriesDemo地址**](https://github.com/yangKJ/KJCategories)
- 再附上一个网络基础库[**RxNetworksDemo地址**](https://github.com/yangKJ/RxNetworks)
- 喜欢的老板们可以点个星🌟，谢谢各位老板！！！

✌️.
