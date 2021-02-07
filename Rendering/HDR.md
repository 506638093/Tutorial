在LDR中我们的颜色和亮度都被限制在了0-1之间。但我们在通过光照计算之后亮度是可能超过1的。这样被强制截断就会出现大量发白的区域。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_166017066f1c8d23.png)  
这损失了很多的细节，使场景看起来非常假。

解决这个问题的一个方案是减小光源的强度从而保证场景内没有一个片段亮于1.0。然而这并不是一个好的方案。一个更好的方案是让颜色暂时超过1.0，然后将其转换至0.0到1.0的区间内，从而防止损失细节。

显示器被限制为只能显示值为0.0到1.0间的颜色，但是在光照方程中却没有这个限制。通过使片段的颜色超过1.0，我们有了一个更大的颜色范围，这也被称作**HDR**(High Dynamic Range, 高动态范围)。有了HDR，亮的东西可以变得非常亮，暗的东西可以变得非常暗，而且充满细节。

HDR原本是用在相机拍摄技术中：  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1660174656645ea5.png)  
这与我们眼睛工作的原理非常相似，当光线很弱的啥时候，人眼会自动调整从而使过暗和过亮的部分变得更清晰。这就是HDR渲染的基础。

## 浮点帧缓冲
默认的帧缓冲默认一个颜色分量只占用8位(bits)。当使用一个使用32位每颜色分量的浮点帧缓冲时(使用GL_RGB32F 或者GL_RGBA32F)，我们需要四倍的内存来存储这些颜色。所以除非你需要一个非常高的精确度，32位不是必须的，使用GLRGB16F就足够了。

有了一个带有浮点颜色缓冲的帧缓冲，我们可以放心渲染场景到这个帧缓冲中。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_166017a766594f00.png)  
很明显，在隧道尽头的强光的值被约束在1.0，因为一大块区域都是白色的，过程中超过1.0的地方损失了所有细节。因为我们直接转换HDR值到LDR值，这就像我们根本就没有应用HDR一样。为了修复这个问题我们需要做的是无损转化所有浮点颜色值回0.0-1.0范围中。我们需要应用到色调映射。

## Tone Mapping

1. 最简单的色调映射算法是Reinhard色调映射，它涉及到分散整个HDR颜色值到LDR颜色值上，所有的值都有对应。Reinhard色调映射算法平均得将所有亮度值分散到LDR上。我们将Reinhard色调映射应用到之前的片段着色器上，并且为了更好的测量加上一个Gamma校正过滤(包括SRGB纹理的使用)：
```c
void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;

    // Reinhard色调映射
    vec3 mapped = hdrColor / (hdrColor + vec3(1.0));
    // Gamma校正
    mapped = pow(mapped, vec3(1.0 / gamma));

    color = vec4(mapped, 1.0);
}   
```
有了Reinhard色调映射的应用，我们不再会在场景明亮的地方损失细节。当然，这个算法是倾向明亮的区域的，暗的区域会不那么精细也不那么有区分度。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_166017db60774ae5.png)  

2. 另外一种有趣的色调映射是使用曝光(Exposure)。HDR图片包含在不同曝光等级的细节。如果我们有一个场景要展现日夜交替，我们当然会在白天使用低曝光，在夜间使用高曝光，就像人眼调节方式一样。有了这个曝光参数，我们可以去设置可以同时在白天和夜晚不同光照条件工作的光照参数，我们只需要调整曝光参数就行了。

一个简单的曝光色调映射算法会像这样：
```c
uniform float exposure;

void main()
{             
    const float gamma = 2.2;
    vec3 hdrColor = texture(hdrBuffer, TexCoords).rgb;

    // 曝光色调映射
    vec3 mapped = vec3(1.0) - exp(-hdrColor * exposure);
    // Gamma校正 
    mapped = pow(mapped, vec3(1.0 / gamma));

    color = vec4(mapped, 1.0);
}  
```
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_166018229dbad8f3.png)  
这样高曝光可以保留更多的暗部细节，低曝光可以保留更多的量部细节。
有一些技巧被称作自动曝光调整(Automatic Exposure Adjustment)或者叫人眼适应(Eye Adaptation)技术，它能够检测前一帧场景的亮度并且缓慢调整曝光参数模仿人眼使得场景在黑暗区域逐渐变亮或者在明亮区域逐渐变暗。

到这里，HDR的核心技术就讲完了。
但却给我们带来一些问题：  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_16601919e767afed.png)  

## 这里我们先来解释一下什么Gamma：
在现实世界中，如果光的强度增加一倍，那么亮度也会增加一倍，这是线性关系。
早期的显示器(阴极射线管CRT)显示图像的时候，电压增加一倍，亮度并不跟着增加一倍。即输出亮度和电压并不是成线性关系的，而是呈亮度增加量等于电压增加量的2.2次幂的非线性关系：  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1660195175c1413e.png)  
这到底是啥意思呢？我们来看张图：  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1660195fe602a580.png) 
*我们明明就亮度增加了一倍，但经过显示器显示出来的亮度却变暗了。*  
因此我们需要进行一次**Gamma矫正**，也即是在反运算一次。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1660197e00747512.png)  
这就是Gamma矫正的由来：  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_16601985c0326452.png)  
**好了问题来了：这里提到的早期的显示器才存在Gamma问题，那么现在都普及液晶显示器了，那Gamma问题就不应该存在了？**
可事实上液晶显示屏的亮度也不是线性的。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1660685f87bfd979.png)  
因此为了统一和兼容，在液晶显示屏我们进行了Gamma调整。
也就是说几乎目前为止（大部分的显示器，HDR显示器除外）都还是存在Gamma问题的。
**再提问：存在Gamma问题就一定需要Gamma矫正吗？Gamma矫正是软件过程还是硬件过程呢？**

## 什么是sRGB呢
我们来看一下sRGB的由来：
我们想要将照相得到的照片放到所有显示器上都能显示正确。显示器可是有Gamma问题的，因此我们得将照片先进行一次Gamma矫正，得到得新照片就是sRGB了。
巧了，sRGB还刚好符合人眼对亮度的感知。我们人眼对暗部的感知更加敏感。而sRGB就刚好用了更大的空间来存储暗部，较少的空间来存储亮部。

## 为什么需要统一到线性空间
引擎中的光照计算(shader)，都是基于线性空间的公式推导出来的，如果输入的数据是gamma空间存储的格式。那么图像最终计算出来的结果一定不是我们想要的（真实物理的结果，但注意这里并没有说它是错的）。
**所有的输入，计算，输出，都能统一在线性空间中，那么结果是最真实的。**

## Unity是如何处理线性空间和Gamma空间的
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1660755d19d47855.png)  
- 如果是Gamma空间，unity对输入和输出不会做任何处理。所以在Gamma空间，我们的颜色贴图标不标记sRGB都无所谓，反正Gamma空间对贴图不会做任何处理。    
- 如果是Linear空间，对于sRGB纹理，Unity在进行采样的时候会自动**<font color=red>去gamma矫正</font>**这一步是在硬件上完成，同时只对RGB通道进行去Gamma，Alpha保持不变。对于非sRGB的纹理则直接采样。最终所有的输出结果再进行一次Gamma矫正。    

unity里面的设置：  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_166075b3628f71ba.png)  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_166075bc662054de.png)  
我们的Normal Map、Light Map默认是线性的，我们的颜色贴图（漫反射贴图应该是sRGB的，但我们的Mask或者噪点图应该是非sRGB的）。
**那么到底什么纹理应该是sRGB,什么是Linear的呢？**
这里有一个原则：凡是美术画出来的图都应该是sRGB的（因为人眼符合sRGB）。凡是通过计算生成出来的贴图都应该是Linear的。

**Alpha混合**
Unity在线性空间下，UI对于带Alpha半透明的贴图，在混合颜色时就会出现错误。
有的同学可能注意到美术出的效果图，最终unity里面拼出来的效果不一样。

**混合公式：**
Gamma空间下的Alpha混合公式：color = (A.rgb * A.a) + (B.rgb * (1 - A.a))
Linear空间下的Alpha混合公式：color = ((A.rgb ^ 2.2 * A.a) + (B.rgb ^ 2.2 * (1 - A.a))) ^（1 / 2.2)

**解决方法一：**
1）所有UI素材取消勾选sRGB选项，得到的混合公式为：color = ((A.rgb * A.a) + (B.rgb * (1 - A.a))) ^（1 / 2.2) ；
2）然后在输出的颜色上做一次pow（2.2），Unity内置管道可以直接用后处理实现，Universal Render Pipeline管道中可以在BlitPass中做处理，原理是差不多的。

缺点：需要申请一张RT，UI上的字体需要特殊处理。

**解决方法二：**
我们只需要在PS上使用线性空间制作，因为PS默认是使用Gamma空间的，转为线性空间，在Unity内部就是统一空间下制作的，就不需要转换了。做法下图所示：  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_16615214f7d20cfa.png)  

但是这个会存在一个问题，PS调出来的颜色会比原本的亮，因为做了转换。所以需要制作UI相关的同事需要一段时间去适应。