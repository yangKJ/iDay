---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第4天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现像素颜色转换滤镜，通过对像素颜色的不同读取方式获取到相应像素颜色，灰度图移除场景中除了黑白灰以外所有的颜色，让整个图像灰度化；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 转成灰度图滤镜
let filter = C7ColorConvert(with: .gray)

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

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: type.rawValue)`

```
/// 颜色通道`RGB`位置转换
public struct C7ColorConvert: C7FilterProtocol {
    
    public enum ColorType: String, CaseIterable {
        case invert = "C7ColorInvert"
        case gray = "C7Color2Gray"
        case bgra = "C7Color2BGRA"
        case brga = "C7Color2BRGA"
        case gbra = "C7Color2GBRA"
        case grba = "C7Color2GRBA"
        case rbga = "C7Color2RBGA"
    }
    
    private let type: ColorType
    
    public var modifier: Modifier {
        return .compute(kernel: type.rawValue)
    }
    
    public init(with type: ColorType) {
        self.type = type
    }
}
```

- 着色器

取出像素rgb值，然后根据对应像素颜色；灰度图则是取所有的颜色分量，将它们加权或平均；

```
// 颜色反转，1 - rgb
kernel void C7ColorInvert(texture2d<half, access::write> outputTexture [[texture(0)]],
                          texture2d<half, access::read> inputTexture [[texture(1)]],
                          uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 outColor(1.0h - inColor.rgb, inColor.a);
    
    outputTexture.write(outColor, grid);
}

// 转灰度图
kernel void C7Color2Gray(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half3 kRec709Luma = half3(0.2126, 0.7152, 0.0722);
    const half gray = dot(inColor.rgb, kRec709Luma);
    const half4 outColor = half4(half3(gray), 1.0h);
    
    outputTexture.write(outColor, grid);
}

kernel void C7Color2BGRA(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 outColor(inColor.bgr, inColor.a);
    
    outputTexture.write(outColor, grid);
}

kernel void C7Color2BRGA(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 outColor(inColor.brg, inColor.a);
    
    outputTexture.write(outColor, grid);
}

kernel void C7Color2GBRA(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 outColor(inColor.gbr, inColor.a);
    
    outputTexture.write(outColor, grid);
}

kernel void C7Color2GRBA(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 outColor(inColor.grb, inColor.a);
    
    outputTexture.write(outColor, grid);
}

kernel void C7Color2RBGA(texture2d<half, access::write> outputTexture [[texture(0)]],
                         texture2d<half, access::read> inputTexture [[texture(1)]],
                         uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 outColor(inColor.rbg, inColor.a);
    
    outputTexture.write(outColor, grid);
}
```

- 权值法

```
const half3 kRec709Luma = half3(0.2126, 0.7152, 0.0722);
const half gray = dot(inColor.rgb, kRec709Luma);
const half4 outColor = half4(half3(gray), 1.0h);
```

- 平均值法

```
const float color = (inColor.r + inColor.g + inColor.b) / 3.0;
const half4 outColor = half4(color, color, color, 1.0h);
```

总结：  
一般由于人眼对不同颜色的敏感度不一样，所以三种颜色值的权重不一样，一般来说绿色最高，红色其次，蓝色最低，最合理的取值分别为Wr ＝ 30％，Wg ＝ 59％，Wb ＝ 11％，所以权值法相对效果更好一点。

### 对照图

|invert|bgra|brga|
|:-:|:-:|:-:|
|![WX20221125-165057.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/79d70e1b392f415ca7905c4a078b61a9~tplv-k3u1fbpfcp-watermark.image?)|![WX20221125-164833.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fc885f7648f842f9893857aab68baf13~tplv-k3u1fbpfcp-watermark.image?)|![WX20221125-165018.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/99c1aab3460c44b398a9c1fed59228f3~tplv-k3u1fbpfcp-watermark.image?)|
|gbra|grba|rbga|
|![WX20221125-165138.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d810f1d30e4f4b3e8dfb37260ced470b~tplv-k3u1fbpfcp-watermark.image?)|![WX20221125-165151.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ad0a88c494d848828826076aee15c59e~tplv-k3u1fbpfcp-watermark.image?)|![WX20221125-165213.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8122337c932f4abc96fc64874a99c40e~tplv-k3u1fbpfcp-watermark.image?)|

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

- 关于颜色转换滤镜介绍与设计到此为止吧。
- 慢慢再补充其他相关滤镜，喜欢就给我点个星🌟吧。
- [**滤镜Demo地址**](https://github.com/yangKJ/Harbeth)，目前包含`100+`种滤镜，同时也支持CoreImage混合使用。
- 再附上一个开发加速库[**KJCategoriesDemo地址**](https://github.com/yangKJ/KJCategories)
- 再附上一个网络基础库[**RxNetworksDemo地址**](https://github.com/yangKJ/RxNetworks)
- 喜欢的老板们可以点个星🌟，谢谢各位老板！！！

✌️.
