---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第24天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现纯色图片效果滤镜，主要就是生成纯色图片；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 纯色滤镜
ImageView.image = C7Color.purple.mt.colorImage(with: CGSize(width: 600, height: 300))
```

### 效果对比图

- 不同参数下效果

|purple|white|systemPink|
|:-:|:-:|:-:|
|![WX20221216-102421.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6e0618b805b249cc8a0d6c6651fa55f0~tplv-k3u1fbpfcp-watermark.image?)|![WX20221216-102432.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d3f6a27725f241e2850822050158ed0b~tplv-k3u1fbpfcp-watermark.image?)|![WX20221216-102946.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/92b49401fff041e59d23332a49ee00e1~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 颜色扩展

根据指定颜色和尺寸生成纯色图片；

```
#if HARBETH_COMPUTE // Compute module
extension Queen where Base: C7Color {
    
    /// Create a solid color image.
    /// - Parameters:
    ///   - color: Indicates the color.
    ///   - size: Indicates the size of the solid color diagram.
    /// - Returns: Solid color graph.
    public func colorImage(with size: CGSize = CGSize(width: 1, height: 1)) -> C7Image? {
        let width  = Int(size.width > 0 ? size.width : 1)
        let height = Int(size.height > 0 ? size.height : 1)
        let texture = Processed.destTexture(width: width, height: height)
        let filter = C7SolidColor.init(color: base)
        let result = try? Processed.IO(inTexture: texture, outTexture: texture, filter: filter)
        return result?.toImage()
    }
}
#endif
```

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7SolidColor")`；

对外开放参数
- `color`: 生成的图片颜色；

```
/// 纯色滤镜
public struct C7SolidColor: C7FilterProtocol, ComputeFiltering {
    
    public var color: C7Color = .white
    
    public var modifier: Modifier {
        return .compute(kernel: "C7SolidColor")
    }
    
    public func setupSpecialFactors(for encoder: MTLCommandEncoder, index: Int) {
        guard let computeEncoder = encoder as? MTLComputeCommandEncoder else { return }
        var factor = Vector4.init(color: color).to_factor()
        computeEncoder.setBytes(&factor, length: Vector4.size, index: index + 1)
    }
    
    public init(color: C7Color = .white) {
        self.color = color
    }
}
```

macOS获取RGBA值出错解决方案：

```
/// Fixed `*** -getRed:green:blue:alpha: not valid for the NSColor Generic Gray Gamma 2.2 Profile colorspace 1 1;
/// Need to first convert colorspace.
/// See: https://stackoverflow.com/questions/67314642/color-not-valid-for-the-nscolor-generic-gray-gamma-when-creating-sktexture-fro
/// - Returns: Color.
func usingColorSpace_sRGB() -> C7Color {
    #if os(macOS)
    let colors: [C7Color] = [
        .white, .black, .gray, .systemPink
    ]
    if colors.contains(base) {
        return base.usingColorSpace(.sRGB) ?? base
    }
    #endif
    return base
}
```

- 着色器

取出RGBA像素即可； 

```
kernel void C7SolidColor(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         device float4 *colorVector [[buffer(0)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 color = half4(*colorVector);
    
    const half4 outColor = half4(color[0], color[1], color[2], color[3]);
    
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
