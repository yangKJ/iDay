---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第21天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现图像单色效果滤镜，将图像转换为单色版本，根据每个像素的亮度进行着色；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 去雾效果滤镜
let filter = C7Monochrome.init(intensity: 0.83, color: .blue)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下效果

|intensity: 0.25, color: .blue|intensity: 0.5, color: .blue|intensity: 0.83, color: .blue|
|:-:|:-:|:-:|
|![WX20221213-104007.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ec80374e04c44f6c9170e8abb161540a~tplv-k3u1fbpfcp-watermark.image?)|![WX20221213-104018.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c47e7a972200476b8d81ead0ca65bb81~tplv-k3u1fbpfcp-watermark.image?)|![WX20221213-104159.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/077d68bcd4734eab9bba72d9f15981d8~tplv-k3u1fbpfcp-watermark.image?)|

|intensity: 0.5, color: .yellow|intensity: 0.5, color: .red|intensity: 0.5, color: .green|
|:-:|:-:|:-:|
|![WX20221213-104042.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8a6377f6706144e3abd347862f63141b~tplv-k3u1fbpfcp-watermark.image?)|![WX20221213-104055.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/280b27213c964c829815e005a37542dc~tplv-k3u1fbpfcp-watermark.image?)|![WX20221213-104144.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d25790aa28624af78ab57ffcb32f9a88~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Monochrome")`，参数因子`[intensity] + RGBAColor(color: color).toRGB()`；

对外开放参数
- `intensity`: 特定颜色取代正常图像颜色的程度，从0.0到1.0，默认值为0.0；
- `color`: 保留配色方案；

```
/// 将图像转换为单色版本，根据每个像素的亮度进行着色
public struct C7Monochrome: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: 0.0, max: 1.0, value: 0.0)
    
    /// The degree to which the specific color replaces the normal image color, from 0.0 to 1.0, with 0.0 as the default.
    @ZeroOneRange public var intensity: Float = range.value
    
    /// Keep the color scheme
    public var color: C7Color = .zero
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Monochrome")
    }
    
    public var factors: [Float] {
        return [intensity] + RGBAColor(color: color).toRGB()
    }
    
    public init(intensity: Float = range.value, color: C7Color = .zero) {
        self.intensity = intensity
        self.color = color
    }
}
```

- 着色器

获取像素亮度值`dot(inColor.rgb, luminanceWeighting)`，然后混合强度`intensity`原始像素和单色值三者获取新的像素颜色rgb； 

```
kernel void C7Monochrome(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         constant float *intensity [[buffer(0)]],
                         constant float *colorR [[buffer(1)]],
                         constant float *colorG [[buffer(2)]],
                         constant float *colorB [[buffer(3)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half3 luminanceWeighting = half3(0.2125, 0.7154, 0.0721);
    const half luminance = dot(inColor.rgb, luminanceWeighting);
    const half4 desat = half4(half3(luminance), 1.0h);
    const half r = desat.r < 0.5 ? (2.0 * desat.r * half(*colorR)) : (1.0 - 2.0 * (1.0 - desat.r) * (1.0 - half(*colorR)));
    const half g = desat.g < 0.5 ? (2.0 * desat.g * half(*colorG)) : (1.0 - 2.0 * (1.0 - desat.g) * (1.0 - half(*colorG)));
    const half b = desat.b < 0.5 ? (2.0 * desat.b * half(*colorB)) : (1.0 - 2.0 * (1.0 - desat.b) * (1.0 - half(*colorB)));
    const half4 outColor = half4(mix(inColor.rgb, half3(r, g, b), half(*intensity)), inColor.a);
    
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
