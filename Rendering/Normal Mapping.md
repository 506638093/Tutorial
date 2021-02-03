我们都知道目前的主流渲染管线下面，游戏都是由三角形组成的世界。
想要在平坦的三角形上制造更多细节，看上去凹凸不平。我们自然就想到了一种通过贴图来改变法线的技术。这种技术统称为凹凸贴图。

法线贴图技术由Bump Mapping（凹凸贴图）技术演变而来。目的是以低代价给与几何体更多丰富的表面信息。这种算法后被进一步改进成了Parallax Mapping（视差贴图）。

## 理论  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165957a84f4587ac.png)  

如果使用表面法向量，物体表面就非常光滑，被光照就是线性变化的，没有明暗差别。
那如果我们在Fragment上对每个像素使用不同的法线，这样微表面就会更多复杂的变化。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165957c63216acae.png)  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165958339ac8c1ca.png)  
我们看到物体表面细节获得极大提升，开销却不大。  

## 工作原理
我们需要为每个像素提供一个法线。就像Diffuse和Specular贴图一样。所以我们使用一张2D纹理来存储法线数据。但我们知道纹理只有颜色信息，所以一张rgb的贴图用来分别存储xyz，但法线的范围是-1到1，因此存储法线数据时需要一次转换
```c
vec3 rgb_normal = normal * 0.5 + 0.5; // 从 [-1,1] 转换至 [0,1]
```
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165958ae3534532f.png)  
存储后的法线贴图是一张偏蓝色（你几乎会发现法线贴图都是偏蓝色的）。这是因为法线大部分时候都是指向Z轴的，因此是一种偏蓝色。


```c
uniform sampler2D normalMap;

void main()
{
    // 从法线贴图范围[0,1]获取法线
    normal = texture(normalMap, fs_in.TexCoords).rgb;
    // 将法线向量转换为范围[-1,1]
    normal = normalize(normal * 2.0 - 1.0);

    [...]
    // 像往常那样处理光照
}
```
计算时，我们从新将法线颜色从0-1重新映射回-1到1。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_16595973333ca0ca.png)  

这里有一个问题：
我们的法线贴图都是指向正Z方向。然而物体的表面不一定指向正Z方向，那么我们按照上面的方法去计算法线，就会得到一个错误的结果。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_165959909f30b9d8.png)  
那么自然想到一个方法就是每个表面都生成法线，同时记录每个表面的起始朝向，如果模型运动我就得记录模型的变换。
试想一下如果模型有大量相同表面但朝向不一样，我们就会得到一张比较大的法线贴图。同时还得追踪每个表面的变换矩阵，这样感觉特别不方便。

## 切线空间  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1659a5a08ad8752b.png)  
如果导出的法线，在切线空间，那么法线就是朝向正Z方向的。切线空间是相对于单个三角形的本地参考空间。
因此我们只需要一个特定的矩阵，将本地到切线空间中的法线向量转成世界或视图坐标，使它们转向到最终的贴图表面的方向（也就是指向正Z方向）。
构造一个这样的矩阵我们只需要三个向量**tangent、bitangent和normal**也就是TBN矩阵。

#### 那么如何计算切线和副切线：  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1659a6f409bd011c.png)  
上图中我们可以看到边E2纹理坐标的不同，E2是一个三角形的边，这个三角形的另外两条边是ΔU2和ΔV2，它们与切线向量T和副切线向量B方向相同。这样我们可以把边E1和E2用切线向量T和副切线向量B的线性组合表示出来（译注：注意T和B都是单位长度，在TB平面中所有点的T、B坐标都在0到1之间，因此可以进行这样的组合）：

```markdown
E1=ΔU1T+ΔV1B
E2=ΔU2T+ΔV2B
```
我们也可以写成这样：

```markdown
(E1x,E1y,E1z)=ΔU1(Tx,Ty,Tz)+ΔV1(Bx,By,Bz)
(E2x,E2y,E2z)=ΔU2(Tx,Ty,Tz)+ΔV2(Bx,By,Bz)
```
E是两个向量位置的差，ΔU和ΔV是纹理坐标的差。然后我们得到两个未知数（切线T和副切线B）和两个等式。你可能想起你的代数课了，这是让我们去接T和B。

上面的方程允许我们把它们写成另一种格式：矩阵乘法

```markdown
[E1xE2xE1yE2yE1zE2z]=[ΔU1ΔU2ΔV1ΔV2][TxBxTyByTzBz]
```
尝试会意一下矩阵乘法，它们确实是同一种等式。把等式写成矩阵形式的好处是，解T和B会因此变得很容易。两边都乘以ΔUΔV的逆矩阵等于：
```markdown
[ΔU1ΔU2ΔV1ΔV2]−1[E1xE2xE1yE2yE1zE2z]=[TxBxTyByTzBz]
```
这样我们就可以解出T和B了。这需要我们计算出delta纹理坐标矩阵的拟阵。我不打算讲解计算逆矩阵的细节，但大致是把它变化为，1除以矩阵的行列式，再乘以它的共轭矩阵。
```markdown
[TxBxTyByTzBz]=1ΔU1ΔV2−ΔU2ΔV1[ΔV2−ΔU2−ΔV1ΔU1][E1xE2xE1yE2yE1zE2z]
```
有了最后这个等式，我们就可以用公式、三角形的两条边以及纹理坐标计算出切线向量T和副切线B。
*其实我们不需要关心怎么计算出切线或者副切线，这些都是美术工具导出。*

#### 代码中如何使用：
在顶点着色器中我们创建TBN矩阵：
```c
void main()
{
   [...]
   vec3 T = normalize(vec3(model * vec4(tangent,   0.0)));
   vec3 B = normalize(vec3(model * vec4(bitangent, 0.0)));
   vec3 N = normalize(vec3(model * vec4(normal,    0.0)));
   mat3 TBN = mat3(T, B, N)
}
```
然后在像素着色器中：
```c
normal = texture(normalMap, fs_in.TexCoords).rgb;
normal = normalize(normal * 2.0 - 1.0);
normal = normalize(fs_in.TBN * normal);
```
因为最后的normal现在在世界空间中了，就不用改变其他像素着色器的代码了。
但这里有个问题：事实上我们的矩阵也应该放到像素着色器中计算。因为我们的顶点着色器没有办法直接就输出矩阵、输出的矩阵经过插值之后无法保证正确性，所以只能输出切线和副切线和法线，最后在片段着色器中重新计算矩阵。

##### 我们还有一种方法：世界空间向量转换到切线空间来计算：
我们使用TBN矩阵的逆矩阵将所有相关的世界空间向量转换到切线空间。TBN的建构还是一样，但我们在将其发送给像素着色器之前先要求逆矩阵：
```c
vs_out.TBN = transpose(mat3(T, B, N));
```
注意，这里我们使用**transpose**函数，而不是**inverse**函数。正交矩阵（每个轴既是单位向量同时相互垂直）的一大属性是一个正交矩阵的置换矩阵与它的逆矩阵相等。这个属性和重要因为逆矩阵的求得比求置换开销大；结果却是一样的。

这样在像素着色器中，我们就不需要对法线向量进行变换：
```c
void main()
{           
    vec3 normal = texture(normalMap, fs_in.TexCoords).rgb;
    normal = normalize(normal * 2.0 - 1.0);

    vec3 lightDir = fs_in.TBN * normalize(lightPos - fs_in.FragPos);
    vec3 viewDir  = fs_in.TBN * normalize(viewPos - fs_in.FragPos);
    [...]
}
```
注：**第二种方法还需要在像素着色器中进行更多的乘法操作，所以为何我们还用第二种方法呢？**

这是因为：将向量从世界空间转换到切线空间有个额外好处，我们可以把所有相关向量在**顶点着色器**中转换到切线空间，不用在像素着色器中做这件事。这是可行的，因为lightPos和viewPos不是每个fragment运行都要改变，对于fs_in.FragPos，我们也可以在顶点着色器计算它的切线空间位置。基本上，不需要把任何向量在像素着色器中进行变换，而第一种方法中就必须放到像素着色器中，因为采样出来的法线向量对于每个像素着色器都不一样。

所以我们的在顶点着色器中代码正确写法是这样：
```c
out VS_OUT {
    vec3 FragPos;
    vec2 TexCoords;
    vec3 TangentLightPos;
    vec3 TangentViewPos;
    vec3 TangentFragPos;
} vs_out;

uniform vec3 lightPos;
uniform vec3 viewPos;

[...]

void main()
{
    [...]
    mat3 TBN = transpose(mat3(T, B, N));
    vs_out.TangentLightPos = TBN * lightPos;
    vs_out.TangentViewPos  = TBN * viewPos;
    vs_out.TangentFragPos  = TBN * vec3(model * vec4(position, 0.0));
}
```

##### 最后一个需要解决的问题：
我们的模型有大量的共享顶点，而这些顶点在不同的面会有不同的切线和副切线。这个问题我们如何来解决呢?而且我们已知顶点的法线和切线为什么还需要副切线呢，这个完全可以计算出来的。
因此我们有了新的方法计算我们的TBN矩阵：
```c
vec3 T = normalize(vec3(model * vec4(tangent, 0.0)));
vec3 N = normalize(vec3(model * vec4(tangent, 0.0)));
// re-orthogonalize T with respect to N
T = normalize(T - dot(T, N) * N);
// then retrieve perpendicular vector B with the cross product of T and N
vec3 B = cross(T, N);

mat3 TBN = mat3(T, B, N)
```
这样稍微花费一些性能开销就能完美解决上述问题，同时我们的模型数据也不需要导出副切线数据。

## unity中如何使用：
```c
Shader "NormalMap"
{
	//属性
	Properties{
		_Diffuse("Diffuse", Color) = (1,1,1,1)
		_MainTex("Base 2D", 2D) = "white"{}
		_BumpMap("Bump Map", 2D) = "bump"{}
		_BumpScale ("Bump Scale", Range(0.1, 30.0)) = 10.0
	}
	//子着色器	
	SubShader
	{
		Pass
		{
			//定义Tags
			Tags{ "RenderType" = "Opaque" }
			CGPROGRAM
			//引入头文件
			#include "Lighting.cginc"
			//定义Properties中的变量
			fixed4 _Diffuse;
			sampler2D _MainTex;
			//使用了TRANSFROM_TEX宏就需要定义XXX_ST
			float4 _MainTex_ST;
			sampler2D _BumpMap;
			float _BumpScale;
 
			//定义结构体：vertex shader阶段输出的内容
			struct v2f
			{
				float4 pos : SV_POSITION;
				//转化纹理坐标
				float2 uv : TEXCOORD0;
				//tangent空间的光线方向
				float3 lightDir : TEXCOORD1;
			};
 
			//定义顶点shader
			v2f vert(appdata_tan v)
			{
				v2f o;
				o.pos = mul(UNITY_MATRIX_MVP, v.vertex);
				//这个宏为我们定义好了模型空间到切线空间的转换矩阵rotation，注意后面有个；
				TANGENT_SPACE_ROTATION;
				//ObjectSpaceLightDir可以把光线方向转化到模型空间，然后通过rotation再转化到切线空间
				o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex));
				//通过TRANSFORM_TEX宏转化纹理坐标，主要处理了Offset和Tiling的改变,默认时等同于o.uv = v.texcoord.xy;
				o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
				return o;
			}
 
			//定义片元shader
			fixed4 frag(v2f i) : SV_Target
			{
				//unity自身的diffuse也是带了环境光，这里我们也增加一下环境光
				fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz * _Diffuse.xyz;
				//直接解出切线空间法线
				float3 tangentNormal = UnpackNormal(tex2D(_BumpMap, i.uv));
				//normalize一下切线空间的光照方向
				float3 tangentLight = normalize(i.lightDir);
				//根据半兰伯特模型计算像素的光照信息
				fixed3 lambert = 0.5 * dot(tangentNormal, tangentLight) + 0.5;
				//最终输出颜色为lambert光强*材质diffuse颜色*光颜色
				fixed3 diffuse = lambert * _Diffuse.xyz * _LightColor0.xyz + ambient;
				//进行纹理采样
				fixed4 color = tex2D(_MainTex, i.uv);
				return fixed4(diffuse * color.rgb, 1.0);
			}
 
			//使用vert函数和frag函数
			#pragma vertex vert
			#pragma fragment frag	
 
			ENDCG
		}
 
	}
		//前面的Shader失效的话，使用默认的Diffuse
		FallBack "Diffuse"
}
```

## 法线的其他用途：
法线贴图除了能增加微表面细节，还可以统一微表面。例如卡渲。  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1659f2dd219de464.png)  
![](https://github.com/506638093/Tutorial/blob/master/Rendering/Textures/attach_1659f345219aa534.png)  