---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第14天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现图像4x4颜色矩阵效果滤镜，通过4x4矩阵对RGBA像素处理；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 绿色通道加倍
let filter = C7ColorMatrix4x4(matrix: Matrix4x4.Color.greenDouble)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

|identity: 原始|sepia: 棕褐色|nostalgic: 怀旧效果|
|:-:|:-:|:-:|
|![WX20221205-094805.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7970938dce994b7a81f60a9147cdab00~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-094822.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/898ec69bb25e4bbeaf743d179ac71117~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-094836.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/392200fb6dad48aa8a5aa1ab48158162~tplv-k3u1fbpfcp-watermark.image?)|
|retroStyle: 复古效果|polaroid: 宝丽来彩色|greenDouble: 绿色通道加倍|
|![WX20221205-094857.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bfe89f1520e44a1381edf73541ced76d~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-094913.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a2f71bcaf3184e8997d37837ad328c6b~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-094954.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eff84d283ee6427cb3c0a513d7c8db2c~tplv-k3u1fbpfcp-watermark.image?)|
|skyblue_turns_green :天蓝色变绿色|gray: 灰度图矩阵|remove_green_blue: 去掉绿色和蓝色|
|![WX20221205-095008.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/debe58fd975e4a8c8cdb77c8affee295~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-095024.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/08bb973d61444dd398c80879709fb301~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-095037.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1424a4e961f74a9a90fca5e56d36f894~tplv-k3u1fbpfcp-watermark.image?)|
|replaced_red_green: 红色绿色对调位置|rgb_to_bgr: 映射RGB到BGR|decreasingOpacity: 调整透明度|
|![WX20221205-095051.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0de740d2b754475596146a16c33359fd~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-095121.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/eaedbb7f6ac74d0583e06f292cbdd5c0~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-095139.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b248c2e75bd744d49d50ac742fa0f2e0~tplv-k3u1fbpfcp-watermark.image?)|
|axix_red_rotate: 围绕红色旋转|axix_green_rotate: 围绕绿色旋转|axix_blue_rotate: 围绕蓝色旋转|
|![WX20221205-095155.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce0f684b49ec4bc5a51ce91548a9f9ca~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-095212.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c824e7a6143478a8948bef845024b21~tplv-k3u1fbpfcp-watermark.image?)|![WX20221205-095226.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/492f280a8e4a4099a0ea235662cd3fe4~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7ColorMatrix4x4")`，参数因子`[intensity] + offset.toFloatArray()`

对外开放参数
- `intensity`: 新转换的颜色取代每个像素原始颜色的程度，默认1；
- `offset`: 每个通道的颜色偏移量；
- `matrix`: 颜色矩阵；

```
/// 4x4 color matrix.
public struct C7ColorMatrix4x4: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: 0.0, max: 1.0, value: 1.0)
    
    /// The degree to which the new transformed color replaces the original color for each pixel, default 1
    @ZeroOneRange public var intensity: Float = range.value
    /// Color offset for each channel.
    public var offset: RGBAColor = .zero
    public var matrix: Matrix4x4
    
    public var modifier: Modifier {
        return .compute(kernel: "C7ColorMatrix4x4")
    }
    
    public var factors: [Float] {
        return [intensity] + offset.toFloatArray()
    }
    
    public func setupSpecialFactors(for encoder: MTLCommandEncoder, index: Int) {
        guard let computeEncoder = encoder as? MTLComputeCommandEncoder else { return }
        var factor = matrix.to_factor()
        computeEncoder.setBytes(&factor, length: Matrix4x4.size, index: index + 1)
    }
    
    public init(matrix: Matrix4x4) {
        self.matrix = matrix
    }
}
```

```
/// RGBA色彩空间中的颜色，在`0 ~ 1`区间内
/// Color in the RGBA color space, from 0 to 1.
public struct RGBAColor {
    
    public static let zero = RGBAColor(red: 0, green: 0, blue: 0, alpha: 0)
    public static let one  = RGBAColor(red: 1, green: 1, blue: 1, alpha: 1)
    
    @ZeroOneRange public var red: Float
    @ZeroOneRange public var green: Float
    @ZeroOneRange public var blue: Float
    @ZeroOneRange public var alpha: Float
    
    public init(red: Float, green: Float, blue: Float, alpha: Float = 1.0) {
        self.red = red
        self.green = green
        self.blue = blue
        self.alpha = alpha
    }
    
    public init(color: C7Color) {
        // See: https://developer.apple.com/documentation/uikit/uicolor/1621919-getred
        let tuple = color.mt.toRGBA()
        self.init(red: tuple.red, green: tuple.green, blue: tuple.blue, alpha: tuple.alpha)
    }
}

extension RGBAColor: Convertible {
    public func toFloatArray() -> [Float] {
        [red, green, blue, alpha]
    }
    
    public func toRGB() -> [Float] {
        [red, green, blue]
    }
}

extension RGBAColor: Equatable {
    
    public static func == (lhs: RGBAColor, rhs: RGBAColor) -> Bool {
        lhs.red == rhs.red &&
        lhs.green == rhs.green &&
        lhs.blue == rhs.blue &&
        lhs.alpha == rhs.alpha
    }
}
```

- 着色器

对RGBA进行矩阵运算，然后加上偏移值offset，按intensity程度获取新的像素颜色；  

```
kernel void C7ColorMatrix4x4(texture2d<half, access::write> outputTexture [[texture(0)]],
                             texture2d<half, access::read> inputTexture [[texture(1)]],
                             constant float *intensity [[buffer(0)]],
                             constant float *redOffset [[buffer(1)]],
                             constant float *greenOffset [[buffer(2)]],
                             constant float *blueOffset [[buffer(3)]],
                             constant float *alphaOffset [[buffer(4)]],
                             constant float4x4 *colorMatrix [[buffer(5)]],
                             uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    const half r = inColor.r;
    const half g = inColor.g;
    const half b = inColor.b;
    const half a = inColor.a;
    
    const half4x4 matrix = half4x4(*colorMatrix);
    const half red   = r * matrix[0][0] + g * matrix[0][1] + b * matrix[0][2] + a * matrix[0][3] + (*redOffset);
    const half green = r * matrix[1][0] + g * matrix[1][1] + b * matrix[1][2] + a * matrix[1][3] + (*greenOffset);
    const half blue  = r * matrix[2][0] + g * matrix[2][1] + b * matrix[2][2] + a * matrix[2][3] + (*blueOffset);
    const half alpha = r * matrix[3][0] + g * matrix[3][1] + b * matrix[3][2] + a * matrix[3][3] + (*alphaOffset);
    
    const half4 color = half4(red, green, blue, alpha);
    const half4 outColor = half(*intensity) * color + (1.0h - half(*intensity)) * inColor;
    
    outputTexture.write(outColor, grid);
}
```

### 4x4矩阵

- 部分4x4颜色矩阵

```
extension Matrix4x4 {
    /// 常见4x4颜色矩阵，考线性代数时刻😪
    /// 第一行的值决定了红色值，第二行决定绿色，第三行蓝色，第四行是透明通道值
    /// Common 4x4 color matrix
    /// See: https://medium.com/macoclock/coreimage-911-color-matrix-4x4-50a7098414f4
    public struct Color { }
}

extension Matrix4x4.Color {
    
    public static let identity = Matrix4x4(values: [
        1.0, 0.0, 0.0, 0.0,
        0.0, 1.0, 0.0, 0.0,
        0.0, 0.0, 1.0, 0.0,
        0.0, 0.0, 0.0, 1.0,
    ])
    
    /// 棕褐色，老照片
    public static let sepia = Matrix4x4(values: [
        0.3588, 0.7044, 0.1368, 0.0,
        0.2990, 0.5870, 0.1140, 0.0,
        0.2392, 0.4696, 0.0912, 0.0,
        0.0000, 0.0000, 0.0000, 1.0,
    ])
    
    /// 怀旧效果
    public static let nostalgic = Matrix4x4(values: [
        0.272, 0.534, 0.131, 0.0,
        0.349, 0.686, 0.168, 0.0,
        0.393, 0.769, 0.189, 0.0,
        0.000, 0.000, 0.000, 1.0,
    ])
    
    /// 复古效果
    public static let retroStyle = Matrix4x4(values: [
        0.50, 0.50, 0.50, 0.0,
        0.33, 0.33, 0.33, 0.0,
        0.25, 0.25, 0.25, 0.0,
        0.00, 0.00, 0.00, 1.0,
    ])
    
    /// 宝丽来彩色
    public static let polaroid = Matrix4x4(values: [
        1.4380, -0.062, -0.062, 0.0,
        -0.122, 1.3780, -0.122, 0.0,
        -0.016, -0.016, 1.4830, 0.0,
        -0.030, 0.0500, -0.020, 1.0,
    ])
    
    /// 绿色通道加倍
    public static let greenDouble = Matrix4x4(values: [
        1.0, 0.0, 0.0, 0.0,
        0.0, 2.0, 0.0, 0.0,
        0.0, 0.0, 1.0, 0.0,
        0.0, 0.0, 0.0, 1.0,
    ])
    
    /// 天蓝色变绿色，天蓝色是由绿色和蓝色叠加
    public static let skyblue_turns_green = Matrix4x4(values: [
        1.0, 0.0, 0.0, 0.0,
        0.0, 1.0, 0.0, 0.0,
        0.0, 0.0, 0.0, 0.0,
        0.0, 0.0, 0.0, 1.0,
    ])
    
    /// 灰度图矩阵，平均值法
    public static let gray = Matrix4x4(values: [
        0.33, 0.33, 0.33, 0.0,
        0.33, 0.33, 0.33, 0.0,
        0.33, 0.33, 0.33, 0.0,
        0.00, 0.00, 0.00, 1.0,
    ])
    
    /// 去掉绿色和蓝色
    public static let remove_green_blue = Matrix4x4(values: [
        1.0, 0.0, 0.0, 0.0,
        0.0, 0.0, 0.0, 0.0,
        0.0, 0.0, 0.0, 0.0,
        0.0, 0.0, 0.0, 1.0,
    ])
    
    /// 红色绿色对调位置
    public static let replaced_red_green = Matrix4x4(values: [
        0.0, 1.0, 0.0, 0.0,
        1.0, 0.0, 0.0, 0.0,
        0.0, 0.0, 1.0, 0.0,
        0.0, 0.0, 0.0, 1.0,
    ])
    
    /// 白色剪影
    /// In case you have to produce a white silhouette you need to supply data to the last column of the color matrix.
    public static let white_silhouette = Matrix4x4(values: [
        0.0, 0.0, 0.0, 1.0,
        0.0, 0.0, 0.0, 1.0,
        0.0, 0.0, 0.0, 1.0,
        0.0, 0.0, 0.0, 1.0,
    ])
    
    /// maps RGB to BGR (rows permuted)
    public static let rgb_to_bgr = Matrix4x4(values: [
        0.22, 0.22, 0.90, 0.0,
        0.11, 0.70, 0.44, 0.0,
        0.90, 0.11, 0.11, 0.0,
        0.00, 0.00, 0.00, 1.0
    ])
    
    /// When you have a premultiplied image, where RGB is multiplied by Alpha, decreasing A value you decrease a whole opacity of RGB.
    /// Thus, any underlying layer becomes partially visible from under our translucent image.
    /// - Parameter alpha: Alpha, 0 ~ 1
    public static func decreasingOpacity(_ alpha: Float) -> Matrix4x4 {
        var matrix = Matrix4x4.Color.identity
        matrix.values[15] = min(1.0, max(0.0, alpha))
        return matrix
    }
    
    /// Rotates the color matrix by alpha degrees clockwise about the red component axis.
    /// - Parameter angle: rotation degree.
    /// - Returns: 4x4 color matrix.
    public static func axix_red_rotate(_ angle: Float) -> Matrix4x4 {
        var matrix = Matrix4x4.Color.identity
        let radians = angle * Float.pi / 180.0
        matrix.values[5] = cos(radians)
        matrix.values[6] = sin(radians)
        matrix.values[9] = -sin(radians)
        matrix.values[10] = cos(radians)
        return matrix
    }
    
    /// Rotates the color matrix by alpha degrees clockwise about the green component axis.
    /// - Parameter angle: rotation degree.
    /// - Returns: 4x4 color matrix.
    public static func axix_green_rotate(_ angle: Float) -> Matrix4x4 {
        var matrix = Matrix4x4.Color.identity
        let radians = angle * Float.pi / 180.0
        matrix.values[0] = cos(radians)
        matrix.values[2] = -sin(radians)
        matrix.values[7] = sin(radians)
        matrix.values[9] = cos(radians)
        return matrix
    }
    
    /// Rotates the color matrix by alpha degrees clockwise about the blue component axis.
    /// - Parameter angle: rotation degree.
    /// - Returns: 4x4 color matrix.
    public static func axix_blue_rotate(_ angle: Float) -> Matrix4x4 {
        var matrix = Matrix4x4.Color.identity
        let radians = angle * Float.pi / 180.0
        matrix.values[0] = cos(radians)
        matrix.values[1] = sin(radians)
        matrix.values[4] = -sin(radians)
        matrix.values[5] = cos(radians)
        return matrix
    }
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
