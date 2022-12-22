---
theme: cyanosis
---
开启掘金成长之旅！这是我参与「掘金日新计划 · 12 月更文挑战」的第26天，[点击查看活动详情](https://juejin.cn/post/7167294154827890702)

本案例的目的是理解如何用Metal实现基于色温调整白平衡效果滤镜，主要就是消除或减轻日光下偏蓝和白炽灯下偏黄，简单讲把应该是白色的调成白色或接近白色，不使其严重偏色；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 白平衡滤镜
let filter = C7WhiteBalance.init(temperature: 4000, tint: -200)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下效果

|temperature: 4000, tint: -200|temperature: 4000, tint: 0|temperature: 4000, tint: 200|
|:-:|:-:|:-:|
|![WX20221220-092550.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/afa1f02415314843b8bcec683dbb8338~tplv-k3u1fbpfcp-watermark.image?)|![WX20221220-092642.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f7e7fa50b9cc4a8cbd8b8c7cd8b1d274~tplv-k3u1fbpfcp-watermark.image?)|![WX20221220-092602.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/567f8b49e2064ef780feb8f0340bf0eb~tplv-k3u1fbpfcp-watermark.image?)|
|temperature: 7000, tint: -200|temperature: 7000, tint: 0|temperature: 7000, tint: 100|
|![WX20221220-092657.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/03af8d1d8ef743f6973f2c43d7ef6b81~tplv-k3u1fbpfcp-watermark.image?)|![WX20221220-092708.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0d92633b9e0840439b8ae0c8d5868dde~tplv-k3u1fbpfcp-watermark.image?)|![WX20221220-092726.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/39bfd4100ade4c6ebdefc64b9e5a3fe9~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7WhiteBalance")`，参数因子`[temperature, tint]`；

对外开放参数
- `temperature`: 调整图像的温度，4000的值非常凉爽，7000非常温暖；
- `tint`: 调整图像的色调，-200的值是非常绿色的，200是非常粉红色的；

```
/// 白平衡
public struct C7WhiteBalance: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: 4000, max: 7000, value: 5000)
    
    /// The tint to adjust the image by. A value of -200 is very green and 200 is very pink.
    public var tint: Float = 0
    /// The temperature to adjust the image by, in ºK. A value of 4000 is very cool and 7000 very warm.
    /// Note that the scale between 4000 and 5000 is nearly as visually significant as that between 5000 and 7000.
    public var temperature: Float = range.value
    
    public var modifier: Modifier {
        return .compute(kernel: "C7WhiteBalance")
    }
    
    public var factors: [Float] {
        return [temperature, tint]
    }
    
    public init(temperature: Float = range.value, tint: Float = 0) {
        self.temperature = temperature
        self.tint = tint
    }
}
```

- 着色器

将图像从RGB空间转换到YIQ空间的rgb值，控制蓝色值处于(-0.5226...0.5226)区间之间，再将YIQ空间颜色转换为RGB空间颜色rgb，对比暖色常量warm获取到新的rgb，最后混色原色和新的rgb和图像色温得到最终的像素rgb； 

```
kernel void C7WhiteBalance(texture2d<half, access::write> outputTexture [[texture(0)]],
                           texture2d<half, access::read> inputTexture [[texture(1)]],
                           constant float *temperature_ [[buffer(0)]],
                           constant float *tint [[buffer(1)]],
                           uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half3x3 RGBtoYIQ = half3x3({0.299, 0.587, 0.114}, {0.596, -0.274, -0.322}, {0.212, -0.523, 0.311});
    const half3x3 YIQtoRGB = half3x3({1.000, 0.956, 0.621}, {1.000, -0.272, -0.647}, {1.000, -1.105, 1.702});
    
    half3 yiq = RGBtoYIQ * inColor.rgb;
    yiq.b = clamp(yiq.b + half(*tint/100) * 0.5226 * 0.1, -0.5226, 0.5226);
    const half3 rgb = YIQtoRGB * yiq;
    
    const half3 warm = half3(0.93, 0.54, 0.0);
    const half r = rgb.r < 0.5 ? (2.0 * rgb.r * warm.r) : (1.0 - 2.0 * (1.0 - rgb.r) * (1.0 - warm.r));
    const half g = rgb.g < 0.5 ? (2.0 * rgb.g * warm.g) : (1.0 - 2.0 * (1.0 - rgb.g) * (1.0 - warm.g));
    const half b = rgb.b < 0.5 ? (2.0 * rgb.b * warm.b) : (1.0 - 2.0 * (1.0 - rgb.b) * (1.0 - warm.b));
    
    half temperature = half(*temperature_);
    temperature = temperature < 5000 ? 0.0004 * (temperature - 5000) : 0.00006 * (temperature - 5000);
    const half4 outColor = half4(mix(rgb, half3(r, g, b), temperature), inColor.a);
    outputTexture.write(outColor, grid);
}
```

### 色温曲线

- 相同的物体在不同的色温光源下呈现出相同的颜色，这就称为色彩恒常性；
- 白平衡的目的就是使得图像传感器在不同色温光源下拍摄的物体进行一个图像纠正的过程，还原物体本来的颜色。
- 也可以说是在任意色温条件下，图像传感器所拍摄的标准白色经过白平衡的调整，使之成像后仍然为白色。

先看看在不同光源下呈现出来的颜色如下图：

<p align="left">
<img src="https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/67d0288befc04e3a91daa56303b0713d~tplv-k3u1fbpfcp-watermark.image" width="400" hspace="1px">
</p>

流程原理：

- 在各个色温下(2500~7500)拍几张白纸照片，假设拍6张(2500,3500...7500)，可以称作色温照；
- 把色温照进行矫正，具体是对R/G/B通道进行轿正，让偏色的白纸照变成白色，并记录各个通道的矫正参数；
- 判断图像的色温，是在白天,晚上,室内,室外，是烈日还是夕阳，还是在阳光下的沙滩上，或者是在卧室里”暖味”的床头灯下；

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
