---
theme: cyanosis
---
开启掘金成长之旅！这是我参与「掘金日新计划 · 12 月更文挑战」的第28天，[点击查看活动详情](https://juejin.cn/post/7167294154827890702)

本案例的目的是理解如何用Metal实现双边模糊效果滤镜，结合图像的空间邻近度和像素值相似度折中处理，同时考虑空域信息和灰度相似性，达到保边去噪的目的；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// 双边模糊滤镜
let filter = C7BilateralBlur.init(radius: 10)

// 方案1:
ImageView.image = try? BoxxIO(element: originImage, filters: [filter, filter2, filter3]).output()

// 方案2:
ImageView.image = originImage.filtering(filter, filter2, filter3)

// 方案3:
ImageView.image = originImage ->> filter ->> filter2 ->> filter3
```

### 效果对比图

- 不同参数下效果

|radius: 10|radius: 2|radius: 0.5|
|:-:|:-:|:-:|
|![WX20221226-170234.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/13e8ae4b90e947ea8594e293cb1bc774~tplv-k3u1fbpfcp-watermark.image?)|![WX20221226-170257.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0157142b058b4da798215862c0192d73~tplv-k3u1fbpfcp-watermark.image?)|![WX20221226-170321.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d99993dc5cfc4812aa0a5017d26b6cd0~tplv-k3u1fbpfcp-watermark.image?)|

### 实现原理

- 过滤器

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7BilateralBlur")`；

```
/// 双边模糊
public struct C7BilateralBlur: C7FilterProtocol {
    
    public var radius: Float = 1
    public var offect: C7Point2D = C7Point2D.center
    
    public var modifier: Modifier {
        return .compute(kernel: "C7BilateralBlur")
    }
    
    public var factors: [Float] {
        return [radius, offect.x, offect.y]
    }
    
    public init(radius: Float = 1) {
        self.radius = radius
    }
}
```

- 着色器

将高斯滤波中通过各个点到中心点的空间临近度计算的各个权值进行优化，将其优化为空间临近度(四周8个点)计算的权值和像素值相似度计算的权值的乘积，优化后的权值再与图像作卷积运算； 

```
kernel void C7BilateralBlur(texture2d<half, access::write> outputTexture [[texture(0)]],
                            texture2d<half, access::sample> inputTexture [[texture(1)]],
                            constant float *blurRadius [[buffer(0)]],
                            constant float *stepOffsetX [[buffer(1)]],
                            constant float *stepOffsetY [[buffer(2)]],
                            uint2 grid [[thread_position_in_grid]]) {
    const int GAUSSIAN_SAMPLES = 9;
    const float x = float(grid.x);
    const float y = float(grid.y);
    const float width = float(inputTexture.get_width());
    const float height = float(inputTexture.get_height());
    const float2 inCoordinate(x / width, y / height);
    
    int multiplier = 0;
    float2 blurStep;
    float2 singleStepOffset(float(*stepOffsetX * 10) / width, float(*stepOffsetY * 10) / height);
    float2 blurCoordinates[GAUSSIAN_SAMPLES];
    
    for (int i = 0; i < GAUSSIAN_SAMPLES; i++) {
        multiplier = (i - ((GAUSSIAN_SAMPLES - 1) / 2));
        blurStep = float(multiplier) * singleStepOffset;
        blurCoordinates[i] = inCoordinate + blurStep;
    }
    
    half4 centralColor;
    half gaussianWeightTotal;
    half4 sum;
    half4 sampleColor;
    half distanceFromCentralColor;
    half gaussianWeight;
    
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const float distanceNormalizationFactor = float(abs(1 - *blurRadius));
    
    centralColor = inputTexture.sample(quadSampler, blurCoordinates[4]);
    gaussianWeightTotal = 0.18;
    sum = centralColor * 0.18;
    
    sampleColor = inputTexture.sample(quadSampler, blurCoordinates[0]);
    distanceFromCentralColor = min(distance(centralColor, sampleColor) * distanceNormalizationFactor, 1.0);
    gaussianWeight = 0.05 * (1.0 - distanceFromCentralColor);
    gaussianWeightTotal += gaussianWeight;
    sum += sampleColor * gaussianWeight;
    
    sampleColor = inputTexture.sample(quadSampler, blurCoordinates[1]);
    distanceFromCentralColor = min(distance(centralColor, sampleColor) * distanceNormalizationFactor, 1.0);
    gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
    gaussianWeightTotal += gaussianWeight;
    sum += sampleColor * gaussianWeight;
    
    sampleColor = inputTexture.sample(quadSampler, blurCoordinates[2]);
    distanceFromCentralColor = min(distance(centralColor, sampleColor) * distanceNormalizationFactor, 1.0);
    gaussianWeight = 0.12 * (1.0 - distanceFromCentralColor);
    gaussianWeightTotal += gaussianWeight;
    sum += sampleColor * gaussianWeight;
    
    sampleColor = inputTexture.sample(quadSampler, blurCoordinates[3]);
    distanceFromCentralColor = min(distance(centralColor, sampleColor) * distanceNormalizationFactor, 1.0);
    gaussianWeight = 0.15 * (1.0 - distanceFromCentralColor);
    gaussianWeightTotal += gaussianWeight;
    sum += sampleColor * gaussianWeight;
    
    sampleColor = inputTexture.sample(quadSampler, blurCoordinates[5]);
    distanceFromCentralColor = min(distance(centralColor, sampleColor) * distanceNormalizationFactor, 1.0);
    gaussianWeight = 0.15 * (1.0 - distanceFromCentralColor);
    gaussianWeightTotal += gaussianWeight;
    sum += sampleColor * gaussianWeight;
    
    sampleColor = inputTexture.sample(quadSampler, blurCoordinates[6]);
    distanceFromCentralColor = min(distance(centralColor, sampleColor) * distanceNormalizationFactor, 1.0);
    gaussianWeight = 0.12 * (1.0 - distanceFromCentralColor);
    gaussianWeightTotal += gaussianWeight;
    sum += sampleColor * gaussianWeight;
    
    sampleColor = inputTexture.sample(quadSampler, blurCoordinates[7]);
    distanceFromCentralColor = min(distance(centralColor, sampleColor) * distanceNormalizationFactor, 1.0);
    gaussianWeight = 0.09 * (1.0 - distanceFromCentralColor);
    gaussianWeightTotal += gaussianWeight;
    sum += sampleColor * gaussianWeight;
    
    sampleColor = inputTexture.sample(quadSampler, blurCoordinates[8]);
    distanceFromCentralColor = min(distance(centralColor, sampleColor) * distanceNormalizationFactor, 1.0);
    gaussianWeight = 0.05 * (1.0 - distanceFromCentralColor);
    gaussianWeightTotal += gaussianWeight;
    sum += sampleColor * gaussianWeight;
    
    const half4 outColor = sum / gaussianWeightTotal;
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
