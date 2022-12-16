---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第9天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现调节图片角度滤镜，通过修改画布大小，取出旋转之后的坐标点像素来达到旋转效果；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 旋转滤镜
let filter = C7Rotate.init(angle: 180)

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

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Rotate")`，参数因子`[Degree(value: angle).radians]`

对外开放参数
- `angle`: 角度旋转，单位是度。

```
/// 旋转
public struct C7Rotate: C7FilterProtocol {
    
    /// Angle to rotate, unit is degree
    public var angle: Float
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Rotate")
    }
    
    public var factors: [Float] {
        return [Degree(value: angle).radians]
    }
    
    public func outputSize(input size: C7Size) -> C7Size {
        return mode.rotate(angle: Degree(value: angle).radians, size: size)
    }
    
    private var mode: ShapeMode = .fitSize
    
    public init(mode: ShapeMode = .fitSize, angle: Float = 0) {
        self.angle = angle
        self.mode = mode
    }
}
```

将角度转换成弧度供着色器使用，

```
public struct Degree {
    
    public let value: Float
    
    public var radians: Float {
        return Float(value * Float.pi / 180.0)
    }
}

// MARK - Negative Degrees
public prefix func -(degree: Degree) -> Degree {
    return Degree(value: -1 * degree.value)
}
```

计算旋转之后的尺寸

```
public func rotate(angle: Float, size: C7Size) -> C7Size {
    switch self {
    case .none:
        return size
    case .fitSize:
        let w = Int(abs(sin(angle) * Float(size.height)) + abs(cos(angle) * Float(size.width)))
        let h = Int(abs(sin(angle) * Float(size.width)) + abs(cos(angle) * Float(size.height)))
        return C7Size(width: w, height: h)
    }
}
```

- 着色器

获取新画布尺寸然后再对坐标点归一化处理，计算超出部分角度，然后算出点坐标(inX, inY)取出对应点的像素，超出部分则用空像素填充；

```
kernel void C7Rotate(texture2d<half, access::write> outputTexture [[texture(0)]],
                     texture2d<half, access::sample> inputTexture [[texture(1)]],
                     constant float *angle [[buffer(0)]],
                     uint2 grid [[thread_position_in_grid]]) {
    const float outX = float(grid.x) - outputTexture.get_width() / 2.0f;
    const float outY = float(grid.y) - outputTexture.get_height() / 2.0f;
    const float dd = distance(float2(outX, outY), float2(0, 0));
    const float pi = 3.14159265358979323846264338327950288;
    const float w = inputTexture.get_width();
    const float h = inputTexture.get_height();
    float outAngle = atan(outY / outX);
    if (outX < 0) { outAngle += pi; };
    const float inAngle = outAngle - float(*angle);
    const float inX = (cos(inAngle) * dd + w / 2.0f) / w;
    const float inY = (sin(inAngle) * dd + h / 2.0f) / h;
    
    // Set empty pixel when out of range
    if (inX * w < -1 || inX * w > w + 1 || inY * h < -1 || inY * h > h + 1) {
        outputTexture.write(half4(0), grid);
        return;
    }
    
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const half4 outColor = inputTexture.sample(quadSampler, float2(inX, inY));
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

- 关于旋转图片滤镜介绍与设计到此为止吧。
- 慢慢再补充其他相关滤镜，喜欢就给我点个星🌟吧。
- [**滤镜Demo地址**](https://github.com/yangKJ/Harbeth)，目前包含`100+`种滤镜，同时也支持CoreImage混合使用。
- 再附上一个开发加速库[**KJCategoriesDemo地址**](https://github.com/yangKJ/KJCategories)
- 再附上一个网络基础库[**RxNetworksDemo地址**](https://github.com/yangKJ/RxNetworks)
- 喜欢的老板们可以点个星🌟，谢谢各位老板！！！

✌️.
