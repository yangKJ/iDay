---
theme: cyanosis
---
「这是我参与2022首次更文挑战的第5天，活动详情查看：[2022首次更文挑战](https://juejin.cn/post/7162096952883019783?utm_source=push&utm_medium=web&utm_campaign=jinshijihua02)」

本案例的目的是理解如何用Metal实现LUT颜色查找表滤镜，通过将颜色值存储在一张表中，在需要的时候通过索引在这张表上找到对应的颜色值，将原有色值替换成查找表中的色值;

总结就是一种针对色彩空间的管理和转换技术，LUT 就是一个 RGB 组合到另一个 RGB 组合的映射关系表；

---

### Demo

- [**HarbethDemo地址**](https://github.com/yangKJ/Harbeth)
- [**iDay每日分享文档地址**](https://github.com/yangKJ/iDay)

### 实操代码

```
// LUT查找滤镜
let filter = C7LookupTable.init(image: R.image("lut_abao"))

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

这款滤镜采用并行计算编码器设计`.compute(kernel: "C7LookupTable")`，参数因子`[intensity]`

对外开放参数
- `intensity`: 强度，其实就是调整mix混合平均值。

```
/// LUT映射滤镜
public struct C7LookupTable: C7FilterProtocol {
    
    public let lookupImage: C7Image?
    public let lookupTexture: MTLTexture?
    public var intensity: Float = 1.0
    
    public var modifier: Modifier {
        return .compute(kernel: "C7LookupTable")
    }
    
    public var factors: [Float] {
        return [intensity]
    }
    
    public var otherInputTextures: C7InputTextures {
        return lookupTexture == nil ? [] : [lookupTexture!]
    }
    
    public init(image: C7Image?) {
        self.lookupImage = image
        self.lookupTexture = image?.cgImage?.mt.newTexture()
    }
    
    public init(name: String) {
        self.init(image: R.image(name))
    }
}
```

- 着色器

1、用蓝色值计算正方形的位置，得到quad1和quad2；  
2、根据红色值和绿色值计算对应位置在整个纹理的坐标，得到texPos1和texPos2；  
3、根据texPos1和texPos2读取映射结果newColor1和newColor2，再用蓝色值的小数部分进行mix操作；

```
kernel void C7LookupTable(texture2d<half, access::write> outputTexture [[texture(0)]],
                          texture2d<half, access::read> inputTexture [[texture(1)]],
                          texture2d<half, access::sample> lookupTexture [[texture(2)]],
                          constant float *intensity [[buffer(0)]],
                          uint2 grid [[thread_position_in_grid]]) {
    const half4 inColor = inputTexture.read(grid);
    const half blueColor = inColor.b * 63.0h; // 蓝色部分[0, 63] 共64种
    
    // 通过蓝色计算两个方格quad1，quad2
    half2 quad1;
    quad1.y = floor(floor(blueColor) / 8.0h);
    quad1.x = floor(blueColor) - (quad1.y * 8.0h);
    
    half2 quad2;
    quad2.y = floor(ceil(blueColor) / 8.0h);
    quad2.x = ceil(blueColor) - (quad2.y * 8.0h);
    
    const float A = 0.125;
    const float B = 0.5 / 512.0;
    const float C = 0.125 - 1.0 / 512.0;
    
    float2 texPos1; // 计算颜色(r,b,g)在第一个正方形中对应位置
    texPos1.x = A * quad1.x + B + C * inColor.r;
    texPos1.y = A * quad1.y + B + C * inColor.g;
    
    float2 texPos2;
    texPos2.x = A * quad2.x + B + C * inColor.r;
    texPos2.y = A * quad2.y + B + C * inColor.g;
    
    constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
    const half4 newColor1 = lookupTexture.sample(quadSampler, texPos1);
    const half4 newColor2 = lookupTexture.sample(quadSampler, texPos2);
    
    const half4 newColor = mix(newColor1, newColor2, fract(blueColor));
    const half4 outColor = half4(mix(inColor, half4(newColor.rgb, inColor.a), half(*intensity)));
    
    outputTexture.write(outColor, grid);
}
```

1、通过蓝色计算两个方格quad1，quad2

```
half2 quad1;
quad1.y = floor(floor(blueColor) / 8.0h);
quad1.x = floor(blueColor) - (quad1.y * 8.0h);

half2 quad2;
quad2.y = floor(ceil(blueColor) / 8.0h);
quad2.x = ceil(blueColor) - (quad2.y * 8.0h);

--------------
比如 inColor(0.4, 0.6, 0.2), 先确定第一个方格：
    
inColor.b = 0.2，blueColor = 0.2 * 63 = 12.6
即为第12个，第13个方格，但是我们要计算它坐在行和列，
floor(12.6) = 12, floor(12 / 8.0h) = 1，即第一行；
floor(blueColor) - (quad1.y * 8.0h) = floor(12.6) - (1 * 8) = 4，即第4列；

同理可以算出第二个方格为第1行，第5列
//ceil 向下取整，ceil(12.6) = 13, 
解决跨行时计算问题，比如blueColor = 7.6，则取第7，8个方格，他们不在同一行
```

2、计算映射后颜色所在两个方格的位置的归一化纹理坐标

```
const float A = 0.125;
const float B = 0.5 / 512.0;
const float C = 0.125 - 1.0 / 512.0;

float2 texPos1; // 计算颜色(r,b,g)在第一个正方形中对应位置
texPos1.x = A * quad1.x + B + C * inColor.r;
texPos1.y = A * quad1.y + B + C * inColor.g;

float2 texPos2;
texPos2.x = A * quad2.x + B + C * inColor.r;
texPos2.y = A * quad2.y + B + C * inColor.g;

--------------
(quad1.x * 0.125)表示行归一化的坐标，
(quad1.y * 0.125)表示列归一化的坐标，一共8行，每一行的长度为1/8 = 0.125，一共8列，每一列的长度为1/8 = 0.125；
(inColor.r * 0.125)表示一个方格里红色的位置，因为一个方格长度为0.125，r从0～1；绿色同理；

需要留意的是这里有个0.5/512 和 1.0/512；
0.5/512 是为了取点的中间值，一个点长度为1，总长度512，取点的中间值，即为0.5/512；
1.0/512 是因为计算texPos2.x时，单独对于一个方格来说，是从0～63，所以为63/512，即0.125 - 1.0 / 512；
```

3、计算映射后颜色

```
// 使用GPU采样器对纹理采样，取出LUT基准图上对于的 R G 色值
constexpr sampler quadSampler(mag_filter::linear, min_filter::linear);
const half4 newColor1 = lookupTexture.sample(quadSampler, texPos1);
const half4 newColor2 = lookupTexture.sample(quadSampler, texPos2);
```

4、混合颜色

```
// 线性取一个平均值，mix 方法根据 b 分量进行两个像素值的混合
const half4 newColor = mix(newColor1, newColor2, fract(blueColor));
// mix(x, y, a); 取x,y的线性混合,x(1-a)+ya
const half4 outColor = half4(mix(inColor, half4(newColor.rgb, inColor.a), half(*intensity))); 
```

### LUT图介绍

LUT图是一张512×512大小的图片，分为64个8×8的小区域，每个小区域对应一个B值(0 ~ 255，间隔4)，小区域内的每个像素点对应一组R和G值(0 ~ 255，间隔为4)。

使用时，获取原图某个像素点的值，通过颜色查找，替换为对应的滤镜颜色值。

![lut_abao.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9c8da3ada05428ca6be8cbc4aea1d9d~tplv-k3u1fbpfcp-watermark.image?)

从图可以看出：
- 8x8的方块组成
- 整体上看每个方块左上角从左上往右下由黑变蓝
- 单独每个方块的右上角是红色为主
- 单独每个方块的左下角是绿色为主

这是一个64x64x64颗粒度的LUT设计，总的方格大小为512x512，8x8=64个方格，所以每个方格大小为64x64；

64个方格，每个方格大小为64x64，所以叫做64x64x64颗粒度的设计。因为颜色值的范围为0～255，即256个取值，将256个取值归化到64;

```
LUT(R1,G1,B1) = (R2,G2,B2)
```

从左上到右下(可以想作z方向)，越来越蓝，蓝色值B从0～255，代表用来查找的B，即LUT(R1,G1,B1) = (R2,G2,B2)中的B1;  
每一个方格里，从左往右(x方向)，红色值R从0～255，代表用来查找的R，即LUT(R1,G1,B1) = (R2,G2,B2)中的R1；  
每一个方格里，从上往下(y方向)，绿色值G从0～255，代表用来查找的G，即LUT(R1,G1,B1) = (R2,G2,B2)中的G1；

因为一个颜色分量是0～255，所以一个方格表示的蓝色范围为4，比如最左上的方格蓝色为0～4，
查找时，如果有某个像素的蓝色值在0～4之间，则一定是在第一个方格里查找其映射后的颜色;

Example:

* 查找像素点归一化后的纯蓝色(0,0,1)的映射后的颜色；

- [x] 使用蓝色B定位方格数

```
n = 1(B值) * 63(一共64个方格，从第0个算起) = 63
```
Answer: 定位的方格n是第63个

- [x] 定位在方格里的位置，使用R，G定位位置x,y

```
x = 0(R值) * 63(每个方格大小为 64 * 64) = 0
y = 0(G值) * 63(每个方格大小为 64 * 64) = 0
```
Answer: 方格的(0,0)位置为要定位的x，y

- [x] 定位在整个图中位置

```
Py = floor(n/8) * 64 + y = 7 * 64 + 0 = 448;
Px = [n - floor(n/8)*8] * 64 + x = [63-7*8] * 64 + 0 = 448;
P1 = (448, 448)

其中floor(n/8)代表位置所在行，每一行的长度为64，y为方格里的G定位的位置；
[n - floor(n/8) * 8]代表位置所在列数，每一列的长度为64，x为方格里的R定位的位置；
floor为向下取整(解决跨行时计算问题)，ceil为向上取整。比如2.3, floor(2.3) = 2; ceil(2.3) = 3;
```
Answer: 方格大小为512x512，位置为P = (448, 448), 归一化后为(7/8, 7/8)  
So: 颜色值(0, 0, 1)的位置确实在第63个方格的左上角；

### 查找方式
LUT分为1D和3D，本质的区别在于索引的输出所需要的索引数

用公式形式看看区别，先设置Ri、Gi、Bi为输入值，Ro、Go、Bo为输出值，LUT标准的转换方法为FuncLUT；

- 1D LUT公式  
Ro = FuncLUT(Ri)  
Go = FuncLUT(Gi)  
Bo = FuncLUT(Bi)

从公式可以看出，各个数值之间独立

- 3D LUT公式  
Ro = FuncLUT(Ri, Gi, Bi)  
Go = FuncLUT(Ri, Gi, Bi)  
Bo = FuncLUT(Ri, Gi, Bi)

在3D LUT中，数值之间会互相影响

从公式对比中我们可以看出来，如果在色深为10位的系统中，1D LUT的数据量大概是3x2^10bit，3D LUT就是(3x2^10)^3bit

由此可以看出3D LUT的数据量比1D LUT多了一个指数级，所以3D LUT的精度比1D LUT高了很多，因为3D LUT的数据量太大，所以是通过列举节点的方式进行数据存储；

参考文章：https://www.jianshu.com/p/f054464e1b40

> 备注: 在相机捕获时实时渲染每一帧图片的时候，就会有显著的性能差别，尤其是 iPhone 8 Plus 相机捕获的每一帧大小几乎都是最后几种情况那么大(4032x3024)

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

- 关于LUT查找滤镜介绍与设计到此为止吧。
- 慢慢再补充其他相关滤镜，喜欢就给我点个星🌟吧。
- [**滤镜Demo地址**](https://github.com/yangKJ/Harbeth)，目前包含`100+`种滤镜，同时也支持CoreImage混合使用。
- 再附上一个开发加速库[**KJCategoriesDemo地址**](https://github.com/yangKJ/KJCategories)
- 再附上一个网络基础库[**RxNetworksDemo地址**](https://github.com/yangKJ/RxNetworks)
- 喜欢的老板们可以点个星🌟，谢谢各位老板！！！

✌️.
