## 原理
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cf4c0ecad8fd.png)  

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cf5449af53ff.png)  


## 阴影体生成
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cf87804f72c2.png)

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cf73f4eedaf0.png)  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cf78ac3c8722.png)  

**结论：当计数为0表示非阴影。反之大于0则阴影。**  


![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cf931671414c.png)  

为了避免产生自阴影和Z-fighting  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cfa703750bd5.png)  


## 计算封闭体积
如果基于从眼睛到像素的正向射线检查，理论上来说是没必要计算封闭体积的。
为什么要计算封闭提及呢？我们后面会讲到。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cfca99aa0b7b.jpg)  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cfcebea2bf83.jpg)  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cfd65b165c8a.jpg)  
只需要乘以矢量（Vx，Vy，Vz，0）（其中'V'是从光源到矢量的点p'的向量）来计算通过视图/投影矩阵并应用透视分割，我们只需将该向量乘以视图/投影矩阵，同时向其添加一个零值的“w”分量 。


## 边界检测  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d05cab958761.png)  
三角面片要么直接受该光源照射（该面法向量与入射光线方向夹角大于90小于180°），要么不直接受光源照射（该面法向量与入射光线方向夹角大于等于0°小于等于90°）。  

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d062452a9cc1.png)  
对某三角形A，有邻面B，C，D。如果A面对光源，那么遍历其三个邻面，如果邻面背对光源，那么A与该邻面共享的那条边即是边界。如果其三个邻面也全都面对光源，那么面A的三条边都不属于边界。



## 问题  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653cfed27086981.jpg)  
我们发现当摄像机处于阴影体内部时，我们的计算将失效了。
因此我们想到了一个方法：将射线反过来计算。也被称之为Z-fail。
但这个方法有一些开销：从背面渲染shadow volume,通常会覆盖更多的像素点,其次从上图可以看出,使用z-fail时必须渲染shadow volume的capping部分(前盖后盖)。这就是为什么我们需要计算出封闭阴影体。


## 实现

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d06f647b86be.jpg)  
1、深度和颜色缓冲区  
关闭深度和颜色缓冲区，打开模板。  
2、从背面绘制阴影体，如果深度测试失败，模板+1  
3、从正面绘制阴影体，如果深度测试失败，模板-1
