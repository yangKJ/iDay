---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第15天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本文正在参加[「金石计划 . 瓜分6万现金大奖」](https://juejin.cn/post/7162096952883019783 "https://juejin.cn/post/7162096952883019783")

本案例的目的是理解如何用Metal实现图像4维向量颜色效果滤镜，通过对像素点颜色进行4维向量叠加运算得到新的像素点；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 暖色系
let filter = C7ColorVector4(vector: Vector4.Color.warm)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

|origin: 原始|warm: 暖色系|cool_tone: 冷色系|
|:-:|:-:|:-:|
|![WX20221205-103059.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/42d2e29cfa70434c9c3bac698b1ed88b~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-103110.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2332d5cba83a44ac85c1da0e60608c6b~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-103121.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/80718a2b9e1f4e809dfbdf81f80d7d3b~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7ColorVector4")`

对外开放参数
- `vector`: 4维向量；

```
/// 四维向量颜色
public struct C7ColorVector4: C7FilterProtocol {
    
    public var vector: Vector4
    
    public var modifier: Modifier {
        return .compute(kernel: "C7ColorVector4")
    }
    
    public func setupSpecialFactors(for encoder: MTLCommandEncoder, index: Int) {
        guard let computeEncoder = encoder as? MTLComputeCommandEncoder else { return }
        var factor = vector.to_factor()
        computeEncoder.setBytes(&factor, length: Vector4.size, index: index + 1)
    }
    
    public init(vector: Vector4) {
        self.vector = vector
    }
}
```

- 着色器

对像素点颜色进行4维向量叠加运算`inColor + half4(*vector)`；  

```
kernel void C7ColorVector4(texture2d<half, access::write> outputTexture [[texture(0)]],
                           texture2d<half, access::read> inputTexture [[texture(1)]],
                           constant float4 *vector [[buffer(0)]],
                           uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 outColor = inColor + half4(*vector);
    
    outputTexture.write(outColor, grid);
}
```

### 4维向量

- 部分4维向量

```
extension Vector4 {
    public struct Color { }
}

extension Vector4.Color {
    
    /// 原始
    public static let origin = Vector4(values: [0.0, 0.0, 0.0, 0.0])
    
    /// 暖色，将rgb通道的颜色添加相应的红/绿色值
    /// Warm color, add the color of the rgb channel to the corresponding red/green value.
    public static let warm = Vector4(values: [0.3, 0.3, 0.0, 0.0])
    
    /// 冷色，将rgb通道的颜色添加相应的蓝色值
    /// Cold color, add the color of the rgb channel to the corresponding blue value.
    public static let cool_tone = Vector4(values: [0.0, 0.0, 0.3, 0.0])
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
