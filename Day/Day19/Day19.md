---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第19天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现灰度系数效果滤镜，输入像素rgb进行次方运算获取到新的rgb；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)

### 实操代码

```
// 灰度系数滤镜
let filter = C7Gamma.init(gamma: 3.0)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下颜色混合效果

|gamma: 1.0|gamma: 2.0|gamma: 3.0|
|:-:|:-:|:-:|
|![WX20221209-141951.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08d3d2b3e7cc4407a96446b84f8d4c2a~tplv-k3u1fbpfcp-watermark.image?)|![WX20221209-141938.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0ce8111442554390b50e346dd224776a~tplv-k3u1fbpfcp-watermark.image?)|![WX20221209-142001.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b03dcc53f3ac43a185d9ef01f79b366a~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Gamma")`，参数因子`[gamma]`；

对外开放参数
- `gamma`: 调整后的伽马，从0到3.0，默认值为1.0；

```
/// 灰度系数
public struct C7Gamma: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: 0.0, max: 3.0, value: 1.0)
    
    /// The adjusted gamma, from 0 to 3.0, with a default of 1.0
    public var gamma: Float = range.value
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Gamma")
    }
    
    public var factors: [Float] {
        return [gamma]
    }
    
    public init(gamma: Float = range.value) {
        self.gamma = gamma
    }
}
```

- 着色器

对输入像素rgb进行次方运算`pow(inColor.rgb, half3(*gamma))`，获取到新的rgb值； 

```
kernel void C7Gamma(texture2d<half, access::write> outputTexture [[texture(0)]],
                    texture2d<half, access::read> inputTexture [[texture(1)]],
                    constant float *gamma [[buffer(0)]],
                    uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 outColor = half4(pow(inColor.rgb, half3(*gamma)), inColor.a);
    
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
