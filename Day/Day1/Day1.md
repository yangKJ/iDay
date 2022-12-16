---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第2天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

**本文正在参加[「金石计划 . 瓜分6万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")**

本案例的目的是理解如何用Metal实现行列分屏滤镜，将图片内容画布切分成行列图；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码：

```
// 分成两行两列
let filter = C7Storyboard.init(ranks: 2)

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

### 实现原理：

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Storyboard")`，参数因子`[Float(N)]`

对外开放参数：
- `N`: 分为`N x N`个屏幕，也就是N行N列；

```
/// 分镜滤镜
public struct C7Storyboard: C7FilterProtocol {

    /// 分为`N x N`个屏幕
    public var N: Int = 2

    public var modifier: Modifier {
        return .compute(kernel: "C7Storyboard")
    }

    public var factors: [Float] {
        return [Float(N)]
    }
    
    public init(ranks: Int = 2) {
        self.N = ranks
    }
}
```

- 着色器

对坐标点归一化处理，然后采用`fmod`函数，取模后的余数再x个数，最后取对应点像素。

```
kernel void C7Storyboard(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::sample> inputTexture [[texture(1)]],
                         constant float *few [[buffer(0)]],
                         uint2 grid [[thread_position_in_grid]]) {
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const float x = float(grid.x) / outputTexture.get_width();
    const float y = float(grid.y) / outputTexture.get_height();
    const float2 textureCoordinate = float2(x, y);
    const int N = int(*few);
    const float2 uv = fmod(textureCoordinate, 1.0 / N) * N;

    const half4 outColor = inputTexture.sample(quadSampler, uv);
    outputTexture.write(outColor, grid);
}
```

### 多滤镜联动

<p align="left">
<img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae83280ff32340a889d7d4a61d0af8f6~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp" width="250" hspace="1px">
</p>

- 运算符链式处理

```swift
/// 1.转换成BGRA
let filter1 = C7ColorConvert(with: C7ColorConvert.ColorType.bgra)

/// 2.调整颗粒度
var filter2 = C7Granularity()
filter2.grain = 0.8

/// 3.调整白平衡
var filter3 = C7WhiteBalance()
filter3.temperature = 5555

/// 4.调整高光阴影
var filter4 = C7HighlightShadow()
filter4.shadows = 0.4
filter4.highlights = 0.5

/// 5.组合操作，获取结果
filterImageView.image = originImage ->> filter1 ->> filter2 ->> filter3 ->> filter4
```

-----

<p align="left">
<img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6f454038a958434da8bc26fc3aa1486a~tplv-k3u1fbpfcp-zoom-in-crop-mark:1304:0:0:0.awebp" width="250" hspace="1px">
</p>

- 组合操作

```swift
/// 1.转换成RBGA
let filter1 = C7ColorConvert(with: C7ColorConvert.ColorType.rbga)

/// 2.调整颗粒度
var filter2 = C7Granularity()
filter2.grain = 0.8

/// 3.配置灵魂效果
var filter3 = C7SoulOut()
filter3.soul = 0.7

/// 4.组合操作
let group: [C7FilterProtocol] = [filter1, filter2, filter3]

/// 5.获取结果
filterImageView.image = try? originImage.makeGroup(filters: group)
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

- 关于分镜滤镜介绍与设计到此为止吧。
- 慢慢再补充其他相关滤镜，喜欢就给我点个星🌟吧。
- [****滤镜Demo地址****](https://github.com/yangKJ/Harbeth)，目前包含`100+`种滤镜，同时也支持CoreImage混合使用。
- 再附上一个开发加速库[****KJCategoriesDemo地址****](https://github.com/yangKJ/KJCategories)
- 再附上一个网络基础库[****RxNetworksDemo地址****](https://github.com/yangKJ/RxNetworks)
- 喜欢的老板们可以点个星🌟，谢谢各位老板！！！

✌️.
