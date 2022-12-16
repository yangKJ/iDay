---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第16天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本文正在参加[「金石计划 . 瓜分6万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")

本案例的目的是理解如何用Metal实现调整对比度效果滤镜，调整对比度就是在保证平均亮度不变的情况下，扩大或缩小亮的点和暗的点之间的差异；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 对比度
let filter = C7Contrast.init(contrast: 2.0)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下对比度效果

|0.5|1.5|2.0|
|:-:|:-:|:-:|
|![WX20221205-104121.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/19bdfc6a6c094e22bb22a1d3ba8db46b~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-104138.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9ed9636c8e1441ddad61b42e87ae8e8d~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-104034.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/93467011c9294b67941206d40b8a35dd~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Contrast")`，参数因子`[contrast]`

对外开放参数
- `contrast`: 调整后的对比度，从0到2.0，默认值为1.0是原始图片；

```
/// 对比度
public struct C7Contrast: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: 0, max: 2.0, value: 1.0)
    
    /// The adjusted contrast, from 0 to 2.0, with a default of 1.0 being the original picture.
    public var contrast: Float = range.value
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Contrast")
    }
    
    public var factors: [Float] {
        return [contrast]
    }
    
    public init(contrast: Float = range.value) {
        self.contrast = contrast
    }
}
```

- 着色器

对每个像素颜色使用对比度系数(contrast)和对比度0(对比度为0的颜色`(rgb = 0.5)`)进行差值；  

```
kernel void C7Contrast(texture2d<half, access::write> outputTexture [[texture(0)]],
                       texture2d<half, access::read> inputTexture [[texture(1)]],
                       constant float *contrast [[buffer(0)]],
                       uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half3 zero = half3(0.5h);// 平均亮度
    const half4 outColor((zero + (inColor.rgb - zero) * half3(*contrast)), inColor.a);
    
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
