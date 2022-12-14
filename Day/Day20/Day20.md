---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第20天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现去雾效果滤镜，类似于UV过滤器；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)

### 实操代码

```
// 去雾效果滤镜
let filter = C7Haze.init(distance: 0.5, slope: 0.5)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下效果

|distance: 0.25, slope: 0.25|distance: 0.25, slope: 0.5|distance: 0.4, slope: 0.5|
|:-:|:-:|:-:|
|![WX20221209-154656.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a4de872023634c9c818c6724edec182c~tplv-k3u1fbpfcp-watermark.image?)|![WX20221209-154718.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c214da21b4934d3294b7e76a85b311b4~tplv-k3u1fbpfcp-watermark.image?)|![WX20221209-154743.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3c2dab9803a5403ab486f083f07315e7~tplv-k3u1fbpfcp-watermark.image?)|


### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Haze")`，参数因子`[distance, slope]`；

对外开放参数
- `distance`: 应用颜色的强度；
- `slope`: 颜色变化量；

```
/// 去雾，类似于UV过滤器
public struct C7Haze: C7FilterProtocol {
    
    /// Strength of the color applied.
    public var distance: Float = 0
    /// Amount of color change.
    public var slope: Float = 0
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Haze")
    }
    
    public var factors: [Float] {
        return [distance, slope]
    }
    
    public init(distance: Float = 0, slope: Float = 0) {
        self.distance = distance
        self.slope = slope
    }
}
```

- 着色器

归一化y乘以颜色变化量，加上强度，得到像素颜色`(inColor - dd * white) / (1.0h - dd)`； 

```
kernel void C7Haze(texture2d<half, access::write> outputTexture [[texture(0)]],
                   texture2d<half, access::read> inputTexture [[texture(1)]],
                   constant float *hazeDistance [[buffer(0)]],
                   constant float *slope [[buffer(1)]],
                   uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half4 white = half4(1.0h);
    const half dd = half(grid.y) / half(inputTexture.get_height()) * half(*slope) + half(*hazeDistance);
    const half4 outColor = half4((inColor - dd * white) / (1.0h - dd));
    
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
