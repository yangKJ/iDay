---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第13天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现连环画滤镜和油画滤镜，将图像处理成连环画和油画效果；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 连环画效果
let filter = C7ComicStrip.init()

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 连环画实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7ComicStrip")`

```
/// 连环画滤镜
public struct C7ComicStrip: C7FilterProtocol {
    
    public var modifier: Modifier {
        return .compute(kernel: "C7ComicStrip")
    }
    
    public init() { }
}
```

- 着色器

获取到红色值`abs(g - b + g + r) * r`绝对值，绿色`abs(b - g + b + r) * r`，蓝色`abs(b - g + b + r) * g`，最后获取到像素颜色；

```
kernel void C7ComicStrip(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    const half r = inColor.r;
    const half g = inColor.g;
    const half b = inColor.b;
    
    const half R = half(abs(g - b + g + r) * r);
    const half G = half(abs(b - g + b + r) * r);
    const half B = half(abs(b - g + b + r) * g);
    
    const half4 outColor = half4(R, G, B, inColor.a);
    
    outputTexture.write(outColor, grid);
}
```

### 油画实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7OilPainting")`，参数因子`[radius, Float(pixel)]`

对外开放参数
- `radius`: 模糊半径；
- `pixel`: 像素颗粒度；

```
/// 油画滤镜
public struct C7OilPainting: C7FilterProtocol {
    
    public var radius: Float = 3.0
    public var pixel: Int = 1
    
    public var modifier: Modifier {
        return .compute(kernel: "C7OilPainting")
    }
    
    public var factors: [Float] {
        return [radius, Float(pixel)]
    }
    
    public init(radius: Float = 3.0, pixel: Int = 1) {
        self.radius = radius
        self.pixel = pixel
    }
}
```

- 着色器

```
kernel void C7OilPainting(texture2d<half, access::write> outputTexture [[texture(0)]],
                          texture2d<half, access::sample> inputTexture [[texture(1)]],
                          constant float *radius [[buffer(0)]],
                          constant float *pixel [[buffer(1)]],
                          uint2 grid [[thread_position_in_grid]]) {
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const float2 size = float2(*pixel) / float2(outputTexture.get_width(), outputTexture.get_height());
    const float2 textureCoordinate = float2(grid) / float2(outputTexture.get_width(), outputTexture.get_height());
    const float r = float(*radius);
    const float n = float((r + 1.0) * (r + 1.0));
    
    float3 m0 = float3(0.0);
    float3 m1 = float3(0.0);
    float3 s0 = float3(0.0);
    float3 s1 = float3(0.0);
    float3 color = float3(0.0);
    
    for (float j = -r; j <= 0.0; ++j)  {
        for (float k = -r; k <= 0.0; ++k)  {
            color = float3(inputTexture.sample(quadSampler, textureCoordinate + float2(k,j) * size).rgb);
            m0 += color;
            s0 += color * color;
        }
    }
    
    for (float j = -r; j <= 0.0; ++j)  {
        for (float k = 0.0; k <= r; ++k)  {
            color = float3(inputTexture.sample(quadSampler, textureCoordinate + float2(k,j) * size).rgb);
            m1 += color;
            s1 += color * color;
        }
    }
    
    half4 outColor = half4(0.0h);
    float min_sigma2 = 100.0;
    m0 /= n;
    s0 = abs(s0 / n - m0 * m0);
    float sigma2 = s0.r + s0.g + s0.b;
    if (sigma2 < min_sigma2) {
        min_sigma2 = sigma2;
        outColor = half4(half3(m0), 1.0h);
    }
    
    m1 /= n;
    s1 = abs(s1 / n - m1 * m1);
    sigma2 = s1.r + s1.g + s1.b;
    if (sigma2 < min_sigma2) {
        min_sigma2 = sigma2;
        outColor = half4(half3(m1), 1.0h);
    }
    
    outputTexture.write(outColor, grid);
}
```

### 效果图

|原图|连环画|油画|
|:-:|:-:|:-:|
||||

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
