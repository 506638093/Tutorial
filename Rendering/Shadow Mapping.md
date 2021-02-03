## Shadow Mapping
我们知道阴影是由于遮挡而导致的光线不足效果。
当光源的光线由于被其他物体遮挡而没有照射到物体时，该物体处于阴影中。阴影为照明场景增添了许多真实感，并使观看者更容易观察对象之间的空间关系。它们使我们的场景和物体更具深度感。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d1d3fde94a45.png)  

阴影贴图是一项被广泛使用，而且相对简单，又极易扩展更多高级算法（Omnidirectional Shadow Maps、Cascaded Shadow Maps）。

## 原理  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d1fd24c4db85.png)  

被光源直接照亮的地方也就变成其他像素的遮挡面。因此我们得出一个结论从光源绘制的深度图也就是我们的阴影贴图了。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d22376fe4fda.png)  

## 实现
1、创建一张深度图，然后关闭颜色，只开启深度。以灯光为相机绘制一遍场景。这样我们就得到了一张深度图。
2、以相机绘制场景。同时开启颜色。
3、在Fragment Shader中，逐像素计算是否处于阴影中。
我们先将uv坐标转换到世界坐标（如果有世界坐标，可省略），再将世界坐标转换到灯光投影空间坐标。最后取出Z值和灯光深度图的Z值做比较。

```c
float ShadowCalculation(vec4 fragPosLightSpace)
{
    vec3 projCoords = fragPosLightSpace.xyz / fragPosLightSpace.w;
    projCoords = projCoords * 0.5 + 0.5;
    float closestDepth = texture(shadowMap, projCoords.xy).r; 
    float currentDepth = projCoords.z;
    float shadow = currentDepth > closestDepth  ? 1.0 : 0.0;
    return shadow;
}
```

## Shadow acne  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d301e1961af8.png)  
我们称之为：Shadow acne（阴影痤疮）

```c
float bias = 0.005; 
float shadow = currentDepth - bias > closestDepth ? 1.0 : 0.0;
```
我们使用一个bias能有效解决这个问题，但如果我们的平面与光源夹角比较大时，仍然也会出现痤疮问题。一种更可靠的算法来解决这个问题：
```c
float bias = max(0.05 * (1.0 - dot(normal, lightDir)), 0.005);  
```
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d3ac11547cea.png)  

## PCF  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d3c930650e15.png)  
因为深度图分辨率问题。我们的阴影边缘锯齿严重。但我们又不太可能创建大分辨率的深度图。
因此我们自然产生了一种柔和阴影的算法：

```c
float shadow = 0.0;
vec2 texelSize = 1.0 / textureSize(shadowMap, 0);
for(int x = -1; x <= 1; ++x)
{
    for(int y = -1; y <= 1; ++y)
    {
        float pcfDepth = texture(shadowMap, projCoords.xy + vec2(x, y) * texelSize).r; 
        shadow += currentDepth - bias > pcfDepth ? 1.0 : 0.0;        
    }    
}
shadow /= 9.0;
```
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1653d3e0668aa479.png)