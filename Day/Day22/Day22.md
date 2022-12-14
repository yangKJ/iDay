---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第22天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现调整透明度效果滤镜，核心就是改变图像像素的透明度值；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)

### 实操代码

```
// 透明度滤镜
let filter = C7Opacity.init(opacity: 0.75)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下效果

|opacity: 0.25|opacity: 0.5|opacity: 0.75|
|:-:|:-:|:-:|
|![WX20221213-111831.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/110099ebe31540468b240e875e388555~tplv-k3u1fbpfcp-watermark.image?)|![WX20221213-111841.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c60cc63f2de74735ada0b010b762980f~tplv-k3u1fbpfcp-watermark.image?)|![WX20221213-111850.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc736fd9dc894be9ae30a038e9ebb3b6~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Opacity")`，参数因子`[opacity]`；

对外开放参数
- `opacity`: 将图像的不透明度从0.0更改为1.0，默认值为1.0；

```
/// 透明度调整，核心就是改变`alpha`
public struct C7Opacity: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: 0.0, max: 1.0, value: 1.0)
    
    /// Change the opacity of an image, from 0.0 to 1.0, with a default of 1.0
    @ZeroOneRange public var opacity: Float = range.value
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Opacity")
    }
    
    public var factors: [Float] {
        return [opacity]
    }
    
    public init(opacity: Float = range.value) {
        self.opacity = opacity
    }
}
```

- 着色器

核心就是改变`inColor.a`； 

```
kernel void C7Opacity(texture2d<half, access::write> outputTexture [[texture(0)]],
                      texture2d<half, access::read> inputTexture [[texture(1)]],
                      device float *opacity [[buffer(0)]],
                      uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 outColor = half4(inColor.rgb, half(*opacity) * inColor.a);
    
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

