# iDay

🫥 Metal Daily Share And Shell Script.

### Day

- 目录列表

| Day | Harbeth Filter Name | 滤镜名 | 链接 |
|:-:|:-|:-|:-|
|Day1|C7Storyboard|分镜|https://juejin.cn/post/7168309341974429704|
|Day2|C7SoulOut|灵魂出窍效果|https://juejin.cn/post/7168306850130051079|
|Day3|C7ConvolutionMatrix3x3|3x3矩阵卷积|https://juejin.cn/post/7168637734326632455|
|Day4|C7ColorConvert|颜色转换|https://juejin.cn/post/7168638424243503111|
|Day5|C7LookupTable|查找滤镜|https://juejin.cn/post/7169096223100829709|
|Day6|C7MeanBlur|均值模糊|https://juejin.cn/post/7169096780368314382|
|Day7|C7Brightness|亮度滤镜|https://juejin.cn/post/7169777856820707336|
|Day8|C7Granularity|胶片颗粒感|https://juejin.cn/post/7169778576076570638|
|Day9|C7Rotate|旋转滤镜|https://juejin.cn/post/7170884104978694181|
|Day10|C7ThresholdSketch|阀值素描|https://juejin.cn/post/7171269095860469768|
|Day11|C7ColorPacking|色彩丢失和模糊效果|https://juejin.cn/post/7171611090831278116|
|Day12|C7Fluctuate|图像波动效果|https://juejin.cn/post/7171986093628194823|
|Day13|C7ComicStrip|连环画滤镜|https://juejin.cn/post/7172371363641229326|
|Day14|C7ColorMatrix4x4|4x4颜色矩阵|https://juejin.cn/post/7173481252895326245|
|Day15|C7ColorVector4|4维向量颜色|https://juejin.cn/post/7173837023147458567|
|Day16|C7Contrast|对比度效果滤镜|https://juejin.cn/post/7174213514515464205|
|Day17|C7Exposure|曝光效果滤镜|https://juejin.cn/post/7174216555822055438|
|Day18|C7FalseColor|虚假颜色混合|https://juejin.cn/post/7174656261533728829|
|Day19|C7Gamma|灰度系数效果|https://juejin.cn/post/7175031825771806780|
|Day20|C7Haze|去雾效果滤镜|https://juejin.cn/post/7176061952346161210|
|Day21|C7Monochrome|图像单色效果|https://juejin.cn/post/7176460309907898426|
|Day22|C7Opacity|透明度|https://juejin.cn/post/7176827523341221946|
|Day23|C7Posterize|海报画效果滤镜||

### Shell

- [图片旋转](https://github.com/yangKJ/iDay/blob/master/Shell/image_rotation.sh)

<details><summary><font size=3>其他sips命令</font></summary>

- 裁剪时固定图片宽度，高度自适应
```
sips -Z 320 iamge_file_name
```

- 裁剪时指定图片宽与高
```
# 裁剪图片为400x300大小
sips -z 400 300 iamge_file_name
```

- 旋转图片
```
sips -r 90 image_file_name
```

- 翻转图片
  - 注：-f支持水平和垂直两种翻转，水平`horizontal`，垂直`vertical`
```
sips -f horizontal image_file_name
```

- 修改图片格式
  - 注：使用-s参数可以修改图片格式为指定值，sips支持jpeg | tiff | png | gif | jp2 | pict | bmp | qtif | psd | sgi | tga共11种格式；
```
sips -s format jpeg input.png --out output.jpg
```

- 获取图片meta信息

```
sips -g pixelWidth -g pixelHeight image_file_name
```

</details>

- [生成ipa文件](https://github.com/yangKJ/iDay/blob/master/Shell/make_ipa.sh)
    - 1.执行Shell脚本
    ```
    sh make_ipa.sh
    ```
    - 2.拖拽.app目录到命令行
    ```
    工程/Products/xxxx.app
    ```
    
    - 3.输入ipa生成目录，不写即生成在桌面

    - 4.输入ipa生成名称
