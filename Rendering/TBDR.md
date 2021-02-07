上世纪90年代，ImgTec和NVIDIA的相互厮杀，最终ImgTec退出桌面市场，但在嵌入式设备市场一家独大。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654241b8b492071.png)  

## 移动端硬件限值
移动端硬件在设计时主要考虑的是功耗问题。
功耗高就意味着发热、耗电、芯片大小。
<font color=#FF0000>**带宽**</font>是功耗的第一杀手。

移动端，屏幕小，分辨率却挺高。渲染每一帧图像都对FrameBuffer访问量惊人的大。在加上手机独特的设计模式，导致显存离显卡较远（不像桌面显卡，显存直接嵌入显卡）。
综上所述：IMR在移动设备上终将不适用。

## 设计思想

移动端的gpu想到了一种化整为零的方法，把巨大的FrameBuffer分解成很多小块，使得每个小块可以被离gpu更近的那个SRAM可以容纳，块的多少取决于你的硬件的SRAM的大小。这样gpu可以分批的一块块的在SRAM上访问framebuffer，一整块都访问好了后整体转移回DRAM上，这样问题就解决了。

为什么这样可以减少带宽？想想，对FrameBuffer现在几乎全部的访问在渲染这一块的时候（test，read，write，blend…）都完全在SRAM上解决，只在最后把这一块整体渲染完了才整体搬回DRAM。这种模式就叫做TBR(tile-based-rendering)。
那么为什么pc不使用tbr？tbdr需要一块块的绘制然后回拷，可以说如果哪一天手机上可以解决带宽产生的功耗问题，或者说sram可以做的足够大了，那么就没有TBDR什么事了。可以简单的认为TBR牺牲了执行效率，但是换来了相对更难解决的带宽功耗。

## 管线
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654244206c6b338.png)  

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654244787188dcb.png)  

在tbr的架构上，是不能够来一个commandbuffer就执行一个的，那是噩梦，因为任何一个commandbuffer都可能影响到到整个FrameBuffer，如果来一个画一个，那么gpu可能会在每一个drawcall上都来回搬迁所有的Tile。这太慢了！

## Tiler
在TBDR的渲染流水线里，几何阶段生成（已经经过裁剪）的多边形或者说三角形参数信息都存放在一个名为Scene Buffer（场景缓存）或者Parameter Buffer（参数缓存）的内存里，理论上里面保存的应该是同一帧画面里的所有三角形信息。  

![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654245af02fd59d.jpg)  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654245d4f2b0edc.jpg)  
*这个阶段也叫Binning Pass。*

在生成Scene Buffer的同时，GPU内部的块元加速器会对这些三角形进行分仓（Binning）或者说筛选（sorting）处理。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures2/attach_16542473f58fc4a1.png)  

这个动作会依照16像素*16像素（取决于具体的GPU实现，像PowerVR PCX2可以是32*32或者32*16，PowerVR SGX 5上是16*16，“新的”PowerVR Series 6/7是32*32，这样的分块被称作块元，即Tile）的大小，将Scene Buffer中位于块元内的所有三角形的指针存放到一个对应的块元缓存（Tile Buffer）中。
对于一个大小为1920x1080的全高清屏幕，用16*16的块元大小进行切分，可以切成大约8100个块元，每个块元在“显存”中都有一个对应的块元缓存，里面存放的就是上面所说的位于块元内的三角形数据指针（指向Scene Buffer对应的三角形数据）。
这些块元缓存存放的数据结构可以称作图元列表（primitive list）。

## Per-Tile Rasterization
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1654247d55353844.jpg)  

前面的Scene Buffer以及Tile Buffers都是存放在GPU外的，而上面步骤中绝大部分操作都是GPU内部的高速缓存上进行的（除了从Scene Buffer中载入该块元的三角形数据外），速度会非常快。

## 思考
- TBR、TBDR分别需要注意什么？
- Alpha Test和Alpha Blend到底消耗如何？
- 全屏后期消耗又如何？
- 如何高效的在移动设备上工作？

## TBR vs TBDR
- TBDR无需对物体做排序，Early-Z也没意义，但TBR最好对物体做排序。
- 大家对处理透明物体都不擅长。但为了尽量减少OverDraw，所以我们需将不透明、透明分开。

## Alpha Test vs Alpha Blend
如果渲染流程中有alpha test/blend/pixel depth write，就会阻断deferred shading，因为这个时候需要执行shading，才能正确进行后续fragment的计算。

- Alpha Test需要执行shading 算出当前fragment的alpha，判断该fragment是否被丢弃。
- Alpha Blend 需要读取framebuffer中当前像素之前的颜色。
- Pixel Depth write会影响到后续fragment的HSR与Depth test。

这三种情况下Pixel Depth Write，因为会影响到后续fragment HSR/Depth，所以这个时候一定要执行该像素的shading，打乱了原先deferred的流程。
而对于AlphaBlend来说，它并不一定要打断deferred shading。
遇到blending的fragment，可以把该fragment像素位置的所有fragment按顺序保留在列表中，等到shading时按顺序计算blend。
但这样就会增加pixel shading的次数。具体的实现还是要参照GPU的实现方式，由于使用TBR，Blend的开销相对比IMR还是降低了很多。

Alpha-Test的情况是和Pixel Depth Write类似，由于Alpha Test失败fragment会被丢弃，如果其开启了DepthWrite，那么就一定要执行shading。因为alpha-test会影响后续fragment的HSR/Z-Test的流程。如果没有开启depth Write，也可以和Blend一样保留后续所有fragment的方式来延迟shading。但是这个时候后续该位置的fragment patch都是不能被移出shading列表的，延迟shading也没有意义了。

## Post Process
- 全屏后处理，这在手机上性能不高，主要就是这些后处理可能在1帧内触发很多次不同的framebuffer之间的bind  unbind，每一次bind unbind都需要对整个framebuffer的一次立即绘制。而tbr的绘制速度是不能和ir相比的，毕竟tile是要一块块从dram拷贝到sram进行渲染的。切换framebuffer在tbdr的架构是性能瓶颈。
- Adreno Vulkan的Flex Render技术，可以自动在TBR和IMR切换。

## 如何高效的在移动设备上工作
- 在IMR模式下，每帧渲染前不需要clear FrameBuffer。但在TBR如果不clear，那么FrameBuffer将会回拷到我们的每个title。同理如果RenderTexture不用了，请记得先clear在销毁。
- 在不透明物体的Fragment shader请不要调用clip或discard。
- 不要在一帧里面频繁切换FrameBuffer。全屏后期处理效率都不高的原因。
- 减少顶点数量、减少顶点信息量。（有的引擎会对顶点数据做压缩）。目的是降低Binning Pass的带宽。
- 基于材质的排序，目的将形同材质或者RenderState的物体合并提交（或者临近提交）。
- 避免大量DrawCall和顶点，如果超过SRAM容量，那将是一场灾难。不管怎么样，更少的顶点，就意味着更少的Binning Pass（更少的带宽）。
- 在移动设备上，CPU、GPU共享内存并且整体供电，意味着他们任何一方如果发热过高，都会被整体降低功率。
- 减少贴图尺寸、压缩贴图、3D开启Mipmap、将贴图合并到Altas。目的都是为了减少带宽和缓存命中率。
- shader减少分支、精确指定数据类型（避免int和float的转换开销，原则是尽量不用int）。

*对于低端Mobile GPU，通常限制性能的是芯片面积，而对高端Mobile GPU，限制性能的则是带宽和发热。
GPU分支：“active mask”的技术，简单来说就是用一个bit mask去判断当前32个thread的branch状态，如果是0，则表示只需要执行false的branch，如果bit mask是2^32-1，则表示只需要执行true的branch，否则就需要对某些thread执行true，同时另一些在执行true的同时等待这些thread，反之亦然，这种情形下一个warp的执行时间就是Max(branch(true))+Max(branch(false))。*