---
theme: cyanosis
---
开启掘金成长之旅！这是我参与「掘金日新计划 · 12 月更文挑战」的第25天，[点击查看活动详情](https://juejin.cn/post/7167294154827890702)

本案例的目的是理解如何用Metal实现自然饱和度效果滤镜，简单讲就是调整图像整体的明亮程度，如调节到较高数值，图像会产生色彩过饱和从而引起图像失真；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 自然饱和度滤镜
let filter = C7Vibrance.init(vibrance: -1.2)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下效果

|vibrance: -1.2|vibrance: -0.5|vibrance: 1.0|
|:-:|:-:|:-:|
|![WX20221220-090820.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/de3d0d43f1154eadbf502de6eb69120b~tplv-k3u1fbpfcp-watermark.image?)|![WX20221220-090653.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/edcfec0e9cda4c7e998ef8eb37cdb2ab~tplv-k3u1fbpfcp-watermark.image?)|![WX20221220-090709.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d60fbaabc0d24ac89aa3b054916810e7~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Vibrance")`，参数因子`[vibrance]`；

对外开放参数
- `vibrance`: 将图像的振动从-1.2更改为1.2，默认值为0.0；

```
/// 自然饱和度
public struct C7Vibrance: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: -1.2, max: 1.2, value: 0.0)
    
    /// Change the vibrance of an image, from -1.2 to 1.2, with a default of 0.0
    public var vibrance: Float = range.value
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Vibrance")
    }
    
    public var factors: [Float] {
        return [vibrance]
    }
    
    public init(vibrance: Float = range.value) {
        self.vibrance = vibrance
    }
}
```

- 着色器

计算出rgb的平均值，获取单通道颜色的最大值，得到饱和度计算后值amt，混合rgb和最大值和饱和度值得到最终的像素rgb； 

```
kernel void C7Vibrance(texture2d<half, access::write> outputTexture [[texture(0)]],
                       texture2d<half, access::read> inputTexture [[texture(1)]],
                       constant float *vibrance [[buffer(0)]],
                       uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half average = (inColor.r + inColor.g + inColor.b) / 3.0h;
    const half mx = max(inColor.r, max(inColor.g, inColor.b));
    const half amt = (mx - average) * (-half(*vibrance) * 3.0h);
    const half4 outColor = half4(mix(inColor.rgb, half3(mx), amt), inColor.a);
    
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
