---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第23天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现海报画效果滤镜，主要就是改变颜色级别数量从而获取到新的像素颜色；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 海报画滤镜
let filter = C7Posterize.init(colorLevels: 2.3)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下效果

|colorLevels: 1.03|colorLevels: 2.3|colorLevels: 5.03|
|:-:|:-:|:-:|
|![WX20221213-112230.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/453f03da31e9420a941c0f08f3ce3cc7~tplv-k3u1fbpfcp-watermark.image?)|![WX20221213-112244.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8015403c270a477491e03236644a81a1~tplv-k3u1fbpfcp-watermark.image?)|![WX20221213-112256.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2958636b80794a17a79e9840555e351d~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7Posterize")`，参数因子`[colorLevels]`；

对外开放参数
- `colorLevels`: 减少图像空间的颜色级别数量；

```
public struct C7Posterize: C7FilterProtocol {
    
    public static let range: ParameterRange<Float, Self> = .init(min: 1.0, max: 255.0, value: 10.0)
    
    /// The number of color levels to reduce the image space to.
    /// This ranges from 1 to 256, with a default of 10
    public var colorLevels: Float = range.value
    
    public var modifier: Modifier {
        return .compute(kernel: "C7Posterize")
    }
    
    public var factors: [Float] {
        return [colorLevels]
    }
    
    public init(colorLevels: Float = range.value) {
        self.colorLevels = colorLevels
    }
}
```

- 着色器

对输入像素x颜色级别数量再加上平均亮度`half4(0.5h)`取整`floor`，再除以颜色级别数量得到最终的像素颜色； 

```
kernel void C7Posterize(texture2d<half, access::write> outputTexture [[texture(0)]],
                        texture2d<half, access::read> inputTexture [[texture(1)]],
                        constant float *colorLevelsPointer [[buffer(0)]],
                        uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    
    const half colorLevels = half(*colorLevelsPointer);
    const half4 outColor = floor((inColor * colorLevels) + half4(0.5h)) / colorLevels;
    
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
