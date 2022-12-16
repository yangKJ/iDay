---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第6天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现均值模糊效果滤镜，均值模糊原理其实很简单通过多个纹理叠加，每个纹理偏移量设置不同达到一点重影效果来实现模糊;

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 均值模糊效果滤镜
let filter = C7MeanBlur.init(radius: 0.5)

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

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7MeanBlur")`，参数因子`[radius]`

对外开放参数
- `radius`: 调整模糊半径，其实就是调整模糊度。

```
/// 均值模糊效果
public struct C7MeanBlur: C7FilterProtocol {
    
    public var radius: Float = 1
    
    public var modifier: Modifier {
        return .compute(kernel: "C7MeanBlur")
    }
    
    public var factors: [Float] {
        return [radius]
    }
    
    public init(radius: Float = 1) {
        self.radius = radius
    }
}
```

- 着色器

对坐标点归一化处理，将颗粒半径缩小百倍，取像素点周边上下左右半径点像素，然后将4个像素点叠加合成以达到模糊效果；

```
kernel void C7MeanBlur(texture2d<half, access::write> outputTexture [[texture(0)]],
                       texture2d<half, access::sample> inputTexture [[texture(1)]],
                       constant float *blurRadius [[buffer(0)]],
                       uint2 grid [[thread_position_in_grid]]) {
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const float2 coordinate = float2(float(grid.x) / outputTexture.get_width(), float(grid.y) / outputTexture.get_height());
    const half radius = half(*blurRadius) / 100.0h;
    
    const half4 sample1 = inputTexture.sample(quadSampler, float2(coordinate.x - radius, coordinate.y - radius));
    const half4 sample2 = inputTexture.sample(quadSampler, float2(coordinate.x + radius, coordinate.y + radius));
    const half4 sample3 = inputTexture.sample(quadSampler, float2(coordinate.x + radius, coordinate.y - radius));
    const half4 sample4 = inputTexture.sample(quadSampler, float2(coordinate.x - radius, coordinate.y + radius));
    
    const half4 outColor = (sample1 + sample2 + sample3 + sample4) / 4.0h;
    outputTexture.write(outColor, grid);
}
```

### 理论
平滑/模糊(Smooth/Blur)是图像处理中最简单和常用的操作，可以给图像预处理时候降低噪声。  
图像平滑处理往往使图像中的边界、轮廓变得模糊，原因是因为图像受到了平均或积分运算，从频率域来考虑，图像模糊的实质是因为其高频分量被衰减

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/619026d174714847beef69f1804a5d53~tplv-k3u1fbpfcp-watermark.image)  
f(i,j)表示一幅图像，第i行j列的像素，h(k,l)是卷积核/卷积算子，k l大小又叫窗口大小，在k l范围内f(i,j)与h(k,l)乘积，各值相加得到一新像素值，输出图像g(i,j)

![WeChata7bb18facd4f5d48864f773fe7a9ff34.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3a36791e57a34914a8831a8c51ad713e~tplv-k3u1fbpfcp-watermark.image)
**卷积过程**：6x6上面是个3x3的窗口，从左向右，从上向下移动，黄色的每个像个像素点值之和取平均值赋给中心红色像素作为它卷积处理之后新的像素值。每次移动一个像素格。

滤波处理分为两大类：线性滤波和非线性滤波

- 线性滤波：方框滤波、均值滤波、高斯滤波  
- 非线性滤波：中值滤波、双边滤波

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

- 关于均值模糊效果滤镜介绍与设计到此为止吧。
- 慢慢再补充其他相关滤镜，喜欢就给我点个星🌟吧。
- [**滤镜Demo地址**](https://github.com/yangKJ/Harbeth)，目前包含`100+`种滤镜，同时也支持CoreImage混合使用。
- 再附上一个开发加速库[**KJCategoriesDemo地址**](https://github.com/yangKJ/KJCategories)
- 再附上一个网络基础库[**RxNetworksDemo地址**](https://github.com/yangKJ/RxNetworks)
- 喜欢的老板们可以点个星🌟，谢谢各位老板！！！

✌️.
