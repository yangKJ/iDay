---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第2天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

**本文正在参加[「金石计划 . 瓜分6万现金大奖」](https://juejin.cn/post/7162096952883019783"https://juejin.cn/post/7162096952883019783")**

本案例的目的是理解如何用Metal实现灵魂出窍滤镜，灵魂出窍效果实现原理是通过两个纹理叠加，根据时间上层纹理做缩放并且不断变化其不透明度来逐渐显现。之前在缓动函数介绍中已经知道如何实现缩放效果，灵魂出窍效果就是在其基础之上再多个纹理对象叠加就能够实现；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)

### 实操代码

```
// 灵魂出窍滤镜
let filter = C7SoulOut.init(soul: 0.7)

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

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7SoulOut")`，参数因子`[soul, maxScale, maxAlpha]`

对外开放参数：
- `soul`: 调整后的灵魂，从0.0到1.0，默认值为0.5
- `maxScale`: 最大灵魂比例
- `maxAlpha`: 灵魂最大透明度

```
/// 灵魂出窍效果
public struct C7SoulOut: C7FilterProtocol {

    public static let soulRange: ParameterRange<Float, Self> = .init(min: 0.0, max: 1.0, value: 0.5)
    
    /// The adjusted soul, from 0.0 to 1.0, with a default of 0.5
    public var soul: Float = soulRange.value
    public var maxScale: Float = 1.5
    public var maxAlpha: Float = 0.5

    public var modifier: Modifier {
        return .compute(kernel: "C7SoulOut")
    }
    
    public var factors: [Float] {
        return [soul, maxScale, maxAlpha]
    }

    public init(soul: Float = soulRange.value, maxScale: Float = 1.5, maxAlpha: Float = 0.5) {
        self.soul = soul
        self.maxScale = maxScale
        self.maxAlpha = maxAlpha
    }
}
```

- 着色器

对坐标点归一化处理，然后采用`最终色 = 基色 * (1 - a) + 混合色 * a`，最后取对应点(soulX, soulY)像素与原像素叠加产生；

```
kernel void C7SoulOut(texture2d<half, access::write> outputTexture [[texture(0)]],
                      texture2d<half, access::sample> inputTexture [[texture(1)]],
                      constant float *soulPointer [[buffer(0)]],
                      constant float *maxScalePointer [[buffer(1)]],
                      constant float *maxAlphaPointer [[buffer(2)]],
                      uint2 grid [[thread_position_in_grid]]) {
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const half4 inColor = inputTexture.read(grid);
    const float x = float(grid.x) / outputTexture.get_width();
    const float y = float(grid.y) / outputTexture.get_height();

    const half soul = half(*soulPointer);
    const half maxScale = half(*maxScalePointer);
    const half maxAlpha = half(*maxAlphaPointer);

    const half alpha = maxAlpha * (1.0h - soul);
    const half scale = 1.0h + (maxScale - 1.0h) * soul;

    const half soulX = 0.5h + (x - 0.5h) / scale;
    const half soulY = 0.5h + (y - 0.5h) / scale;

    // 最终色 = 基色 * (1 - a) + 混合色 * a
    const half4 soulMask = inputTexture.sample(quadSampler, float2(soulX, soulY));
    const half4 outColor = inColor * (1.0h - alpha) + soulMask * alpha;

    outputTexture.write(outColor, grid);
}
```

### 动态效果

<p align="left">
<img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5f2b0a70ab16426fb36054b32c9bc2a9~tplv-k3u1fbpfcp-watermark.image?" width="250" hspace="1px">
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

- 关于灵魂出窍滤镜介绍与设计到此为止吧。
- 慢慢再补充其他相关滤镜，喜欢就给我点个星🌟吧。
- [**滤镜Demo地址**](https://github.com/yangKJ/Harbeth)，目前包含`100+`种滤镜，同时也支持CoreImage混合使用。
- 再附上一个开发加速库[**KJCategoriesDemo地址**](https://github.com/yangKJ/KJCategories)
- 再附上一个网络基础库[**RxNetworksDemo地址**](https://github.com/yangKJ/RxNetworks)
- 喜欢的老板们可以点个星🌟，谢谢各位老板！！！

✌️.
