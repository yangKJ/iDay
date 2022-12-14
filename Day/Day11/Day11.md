---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第11天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现图像包装效果滤镜，用于图像处理色彩丢失和模糊效果；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)

### 实操代码

```
// 色彩丢失和模糊效果
let filter = C7ColorPacking.init(horizontalTexel: 2.5, verticalTexel: 5)

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

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7ColorPacking")`，参数因子`[horizontalTexel, verticalTexel]`

对外开放参数
- `horizontalTexel`: 横向偏移，越大绿色轮廓虚影向右偏移越多；
- `verticalTexel`: 纵向偏移，越大蓝色轮廓虚影向下偏移越多；

```
/// 色彩丢失/模糊效果
public struct C7ColorPacking: C7FilterProtocol {
    
    /// The larger the transverse offset, the more the green contour shadow offset to the right.
    public var horizontalTexel: Float
    /// The larger the vertical offset, the more the blue contour shadow offset downward.
    public var verticalTexel: Float
    
    public var modifier: Modifier {
        return .compute(kernel: "C7ColorPacking")
    }
    
    public var factors: [Float] {
        return [horizontalTexel, verticalTexel]
    }
    
    public init(horizontalTexel: Float = 0, verticalTexel: Float = 0) {
        self.horizontalTexel = horizontalTexel
        self.verticalTexel = verticalTexel
    }
}
```

- 着色器

纹理坐标和偏移均归一化处理，然后获取到上下左右4个纹理偏移坐标，取出对应的纹理红色值，最后得到像素值；  

```
kernel void C7ColorPacking(texture2d<half, access::write> outputTexture [[texture(0)]],
                           texture2d<half, access::sample> inputTexture [[texture(1)]],
                           device float *texelWidthPointer [[buffer(0)]],
                           device float *texelHeightPointer [[buffer(1)]],
                           uint2 grid [[thread_position_in_grid]]) {
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const float width  = outputTexture.get_width();
    const float height = outputTexture.get_height();
    const float texelWidth  = float(*texelWidthPointer / width);
    const float texelHeight = float(*texelHeightPointer / height);
    
    const float2 textureCoordinate = float2(float(grid.x) / width, float(grid.y) / height);
    const float2 upperLeftTextureCoordinate = textureCoordinate + float2(-texelWidth, -texelHeight);
    const float2 upperRightTextureCoordinate = textureCoordinate + float2(texelWidth, -texelHeight);
    const float2 lowerLeftTextureCoordinate = textureCoordinate + float2(-texelWidth, texelHeight);
    const float2 lowerRightTextureCoordinate = textureCoordinate + float2(texelWidth, texelHeight);
    
    half upperLeftIntensity = inputTexture.sample(quadSampler, upperLeftTextureCoordinate).r;
    half upperRightIntensity = inputTexture.sample(quadSampler, upperRightTextureCoordinate).r;
    half lowerLeftIntensity = inputTexture.sample(quadSampler, lowerLeftTextureCoordinate).r;
    half lowerRightIntensity = inputTexture.sample(quadSampler, lowerRightTextureCoordinate).r;
    
    const half4 outColor = half4(upperLeftIntensity, upperRightIntensity, lowerLeftIntensity, lowerRightIntensity);
    
    outputTexture.write(outColor, grid);
}
```

### 效果图

| 横行纵向偏移 | 横向偏移 | 纵向偏移 |
|:-:|:-:|:-:|
|![WX20221129-145900.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/421816bb555f4850908b3d19c8689da9~tplv-k3u1fbpfcp-watermark.image?)|![WX20221129-145951.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7eed5eb975a0433499b1499074b752fd~tplv-k3u1fbpfcp-watermark.image?)|![WX20221129-145924.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae3e81576c0147518810a9ea203bc2b4~tplv-k3u1fbpfcp-watermark.image?)|

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
