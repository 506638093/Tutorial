## 硬件
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165421b8041be400.png)  
内存到显存的带宽速度最慢、显存容量明显小于内存容量。这张图虽然不是最新的硬件架构，但可以基本反映出一个硬件情况，长期来说这种情况也不会有太多改善。

## GPU
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165421efaff03940.png)  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165421f2650efb6d.jpg)  
尽量使 Kernel 代码（每个线程，尤其是同一个 Block 内的线程）具有相同的执行路径（即分支跳转情况尽量相同），以充分利用GPU访存及代码执行方面的并行机制。

## 管线
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_16542210eef4250a.jpg)  
总的来说：大体分为这三个大阶段。

# <font color=#FF0000>应用阶段</font>
- CPU完成、开发人员自己掌控、最终提交给GPU数据。
- Octree、BSP、Node
- 软件遮挡剔除
- 控制渲染流
应用程序阶段，我们的目的就一个：尽量提供给GPU它想要的数据，剔除哪些它不需要关心的数据。

# <font color=#FF0000>几何阶段</font>
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654222f45f141b4.jpg)  
绿色表示开发者可以完全编程控制的部分，虚线外框表示此阶段不是必需的，黄色表示开发者无法完全控制的部分（但可以进行一些配置），紫色表示开发者无法控制的阶段（已经由GPU固定实现）
这里的裁剪是指投影空间的裁剪，将看不到的顶点剔除。

## DrawCall
CPU往命令缓冲区中一个个放入命令，GPU则依次取出执行。在实际的渲染中，GPU的渲染速度往往超过了CPU提交命令的速度，这导致渲染中大部分时间都消耗在了CPU提交Draw Call上。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165422405d321562.jpg)  

## 顶点着色器 Vertex Shaders
- 模型转化与相机转换  
- Lambert光照模型  

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654227aa46b5b0a.jpg)  

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165422858aa372db.jpg)  

顶点着色器是GPU流水线的第一个阶段也是必需的阶段，这一块可以由开发者完全控制。值得一提的是，这顶点着色器中，我们无法创建或销毁任何一个顶点，也无法得到当前处理的这个顶点与其他顶点的关系。因为每次处理顶点都是独立的，不需要额外考虑其他，所以进行这一步速度会相当快。

## 投影
- 正交投影  
- 透视投影  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165422a1ba9e4b11.jpg)  

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165422a93a24a79d.jpg)![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165422aeb459bda0.jpg)  

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165422b5dfaf3c8e.jpg)![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165422b96896b380.jpg)  

## 裁剪
- 在经过投影过程把顶点坐标转换到裁剪空间后，GPU就可以进行裁剪操作了。裁剪操作的目的就是把摄像机看不到的顶点剔除出去，使他们不被渲染到。  
- 背面剔除也发生在这个阶段。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165422cae76f6ff3.png)  
将x、y、z的坐标经过齐次除法变换到[-1,1]的空间。
*特别注意：裁剪阶段是发生在顶点着色器之后，只有经过顶点着色器处理过的顶点才能正确被裁剪。*

## 屏幕映射
尽管GPU已经得到了顶点的x、y坐标，但他们处于[-1,1]区间中的，GPU还需要进行一定的计算才能把他们映射到我们的1920*1080甚至2560*1440的屏幕。得到的新坐标系称为窗口坐标系，虽然只需要两个坐标把顶点投射到屏幕上，但它仍然是三维的，这个多出来的z值就是在上面算出来的深度。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165422f2cd7e431f.jpg)

# <font color=#FF0000>光栅化阶段</font>
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165423490fb5e86c.jpg)

## 图元组装
这个过程做的工作就是把顶点数据收集并组装为简单的基本体（线、点或三角形）。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165423549fe6ecd1.jpg)

## 三角形遍历
这个过程将检验屏幕上的某个像素是否被一个三角形网格所覆盖，被覆盖的区域将生成一个片元。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654235cce174293.jpg)![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654237508a936fe.jpg)![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165423798eedae54.jpg)  
除此以外，GPU还将对覆盖区域的每个像素的深度进行插值计算。因为对于屏幕上的一个像素来说，它可能有着多个三角形的重叠，所以这一步对于后面计算遮挡、半透明等效果有着重要的作用。

## 片元着色器Fragment Shader
- 程序员自己控制着色  
- PBR  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654238f6bd08533.jpg)

## 逐片元操作
主要的工作有两个：对片元进行测试（Test）并进行合并（Merge）
- 裁剪测试  
- 透明度测试  
- 模板测试  
- 深度测试  
- Blend  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654239a9b9fe723.jpg)  
裁剪测试（Scissor Test）允许程序员开设一个裁剪框，只有在裁剪框内的片元才会被显示出来，在裁剪框外的片元皆被剔除。

### 深度测试

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165423b9632ff59f.png)  
*问题：为什么深度测试放在的所有测试的最后面？*

## 双缓冲区
GPU会使用双重缓冲（Double Buffering）的策略，即屏幕上显示前置缓冲（Front Buffer），而渲染好的颜色先被送入后置缓冲（Back Buffer），再替换前置缓冲，以此避免在屏幕上显示正在光栅化的图元。