---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第18天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现虚假颜色效果滤镜，使用图像的亮度在两种用户指定的颜色之间进行混合；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)

### 实操代码

```
// 混合颜色
let filter = C7FalseColor.init(fristColor: .blue, secondColor: .green)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下颜色混合效果

|蓝色～绿色|黄色～棕色|绿色～蓝色|
|:-:|:-:|:-:|
|![WX20221208-135403.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c3ca9df27094b339baa1f1853f02fd4~tplv-k3u1fbpfcp-watermark.image?)|![WX20221208-135622.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/59bfb3cd66b0464ba95f7d83311eb9cc~tplv-k3u1fbpfcp-watermark.image?)|![WX20221208-135721.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/707cfb69413e4ee58af9a7421cf8f4e5~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7FalseColor")`

对外开放参数
- `fristColor`: 第一和第二种颜色分别指定了哪些颜色取代了图像的深色和浅色区域；
- `secondColor`: 第二种颜色；

```
/// 使用图像的亮度在两种用户指定的颜色之间进行混合
/// Uses the luminance of the image to mix between two user-specified colors
public struct C7FalseColor: C7FilterProtocol, ComputeFiltering {
    
    /// The first and second colors specify what colors replace the dark and light areas of the image, respectively.
    public var fristColor: C7Color = .zero
    
    public var secondColor: C7Color = .zero
    
    public var modifier: Modifier {
        return .compute(kernel: "C7FalseColor")
    }
    
    public func setupSpecialFactors(for encoder: MTLCommandEncoder, index: Int) {
        guard let computeEncoder = encoder as? MTLComputeCommandEncoder else { return }
        var fristFactor = Vector3.init(color: fristColor).to_factor()
        computeEncoder.setBytes(&fristFactor, length: Vector3.size, index: index + 1)
        var secondFactor = Vector3(color: secondColor).to_factor()
        computeEncoder.setBytes(&secondFactor, length: Vector3.size, index: index + 2)
    }
    
    public init(fristColor: C7Color = .zero, secondColor: C7Color = .zero) {
        self.fristColor = fristColor
        self.secondColor = secondColor
    }
}
```

- 着色器

计算出输入像素亮度点积`dot(inColor.rgb, luminanceWeighting)`，然后再两种输入颜色当中混合`mix(color1.rgb, color2.rgb, luminance)`获取到rgb； 

```
kernel void C7FalseColor(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         constant float3 *firstVector [[buffer(0)]],
                         constant float3 *secondVector [[buffer(1)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half3 luminanceWeighting = half3(0.2125, 0.7154, 0.0721);
    const half luminance = dot(inColor.rgb, luminanceWeighting);
    const half3 color1 = half3(*firstVector);
    const half3 color2 = half3(*secondVector);
    const half4 outColor = half4(mix(half3(color1.rgb), half3(color2.rgb), half3(luminance)), inColor.a);
    
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
