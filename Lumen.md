# Lumen（Indirect Diffuse）

**总览**

<img src="C:\Typora\Lumen.assets\image-20240122152736340.png" alt="image-20240122152736340"  />

# 加速结构

Lumen里面用到的比较普遍的加速结构有三种，Hi-Z、MDF和GDF。

Hi-Z准确度最高，但却是三种加速结构里面最慢的一种，一般用于屏幕空间Ray Trace。

MDF准确度较高，速度中等，一般在ray即将到达object需要获取物体表面的Lighting Cache时使用。

GDF准确度较低，但拥有最快的tracing速度，GDF由多个低分辨率MDF通过一定计算得出的，GDF用于长距离的ray trace或者是远处物体（相对于屏幕）的ray trace。

![image-20240122154013829](C:\Typora\Lumen.assets\image-20240122154013829.png)

UE一般混用这三种加速结构

![image-20240122153730913](C:\Typora\Lumen.assets\image-20240122153730913.png)

UE加速方案如下

1. 先用Screen Space Trace。基于Hi-Z走50步，如果能Trace到，直接获取结果。
2. 如果Screen Space Trace失败，采用MDF，SDF Trace的距离为1.8米，Trace距离短，也就是只能Trace很近的物体，如果能Trace到返回Mesh ID，可以通过ID获取Surface Cache。
3. 远处物体用GDF，如果需要高精度，先用GDF步进到一定距离再用MDF，如果精度低，直接用GDF一步到位获取结果。
4. 如果GDF也失败了，从CubeMap采样。

## Hi-Z

## SDF

### SDF能干什么

1. SDF就能加速ray求交

每次步进SDF的距离，可以以log2的代价对光线快速求教，如下面右图。

![image-20240122155607277](C:\Typora\Lumen.assets\image-20240122155607277.png)

2. SDF可以快速模拟软阴影

我们可以通过SDF过程中产生的最小角度来模拟光照的最大通量，如下图黄圈，黄圈p3的角度最小，可以近似理解为光能通过这条光路照到o点的覆盖角度只有Θ_3度。

![image-20240122155946588](C:\Typora\Lumen.assets\image-20240122155946588.png)

### MDF（Mesh Distance Fields）

#### MDF原理

UE里面每导入一个Mesh都生成对应的个Mesh Distance Fields，MDF代表对应点到Mesh表面最短的距离，通过这个距离我们可以以log2的代价快速对ray求交。

MDF一般在Mesh Card导入的时候就预计算生成了，因为MDF一般是相对于Mesh本身，而Mesh基本又不会形变，所以MDF一般都不变的。

![image-20240122155327365](C:\Typora\Lumen.assets\image-20240122155327365.png)

生成MDF的时候需要一定的偏移修正，不然会丢失精度

![image-20240122155405967](C:\Typora\Lumen.assets\image-20240122155405967.png)

#### 如何生成MDF？

因为SDF显存占用过高，所以对SDF进行稀疏化处理，在生成MDF的时候只生成很薄的一层SDF，这样可以节省存储量。

![image-20240122160346235](C:\Typora\Lumen.assets\image-20240122160346235.png)

#### MDF LOD

SDF可以做LOD，同时LOD是空间上连续可导的，可以用LOD反向求梯度，也就是说我们用了一个 Uniform 的表达，可以表达出一个无限精度的一个 Mesh ，我既能得到它的面积，又能对它进行快速的求交运算，还能够迅速的求出它连续的这个法线方向。

![image-20240122160812300](C:\Typora\Lumen.assets\image-20240122160812300.png)

LOD和稀疏SDF可以节省40%到60%的空间，对于远处的物体我们可以用Low SDF，近处的物体再用High SDF，这样可以很好的控制内存消耗。

### GDF（Global Distance Fields）

#### 为什么需要GDF？

如果只用给MDF去做Ray Trace速度会很快，精确度也很高，但是如果Mesh特别多的时候，因为MDF只有薄薄一层的SDF，所以远处的ray需要对每个Mesh生成一个SDF再判断哪个SDF最小再使用，这样就需要遍历很多Mesh的MDF，非常耗费性能，所以提出了GDF的概念。

#### GDF原理

GDF是整个场景的SDF，通常GDF的精度都比较低，低精度的GDF是通过场景中每个MDF共同合成的。

GDF可视化

![image-20240122162220557](C:\Typora\Lumen.assets\image-20240122162220557.png)

MDF可视化

![image-20240122162239440](C:\Typora\Lumen.assets\image-20240122162239440.png)

因为GDF代表的是远处没那么高精度的场景，所以我们可以采用Low LOD的MDF去合成，完整的MDF可以用于整个场景的SDF求交加速，比较常用的方法是先用GDF快速Trace一大段距离，再用MDF去做精细的求交。

GDF一般需要实时生成，因为GDF相对于世界空间，而世界空间里面物体随时可能变换位置，所以GDF需要实时生成。

# Mesh Card

Mesh Card是一种结构的统称，在Lumens里面，我们每个导入的物体都会生成一套Mesh Card。Mesh Card 是Mesh的一种属性结构，用于记录数据。

UE里面捕获Mesh Card的方式是通过对6个轴对齐方向（上、下、左、右、前、后）进行光栅化，获取Mesh对应的材质中的Material Attributes（Albedo、Opacity、Normal、Emissive）并存储到Surface Cache上，同时需要捕获的还有对应观察角度的Hit Point，每个面的Hit Point数据需要进行物理地址转换到Surface Cache图集空间中执行采样。

![image-20240122162907587](C:\Typora\Lumen.assets\image-20240122162907587.png)

每个Mesh至少有6个Card，复杂物体可能会生成多个。

![image-20240122162942090](C:\Typora\Lumen.assets\image-20240122162942090.png)

Mesh Card用于记录Material的Attribute和Lighting的Radiance，我们把这两种数据分为Surface Cache和Radiance Cache，而我们最后把数据存入Mesh Card里面。



## Card LOD

大世界中往往存在很多的Mesh，对每个Mesh都进行细致的光栅化是不切实际的，所以对于远距离的Mesh一般采用Card LOD，也就是远处物体不需要生成那么多Card，往往6-8个足够了，这样可以减少光栅化的次数。

![image-20240123102131348](C:\Typora\Lumen.assets\image-20240123102131348.png)

每个 Mesh 的每个 LOD 最多可以有 32 个 Card。

## Surface Cache

Surface Cache受摄像机距离影响，相机离物体比较近的时候Surface Cache分辨率会高一点。

![image-20240122165719982](C:\Typora\Lumen.assets\image-20240122165719982.png)

Surface Cache内的Material属性一般都会经过硬件压缩来减少显存的占用

![image-20240122165818767](C:\Typora\Lumen.assets\image-20240122165818767.png)

Surface Cache属性通常类似于GBuffer的属性，在导入模型的时候就预加载进Surface Cache里面了，Material属性是Surface Cache里面基本不变的数据

![image-20240122164649807](C:\Typora\Lumen.assets\image-20240122164649807.png)

对于较远的物体，在光栅化记录Surface Cache中Material Attribute的时候，可以采用更低分辨率进一步减少光栅化和存储，这就是Surface Mipmap。与传统 Mipmap 存储方式不同，Surface Mipmap 在 Surface Cache 平铺展开存储，因此在采样时需要额外的地址转换。

下面左右分别是同一个Mesh的不同Mipmap

![image-20240123103053265](C:\Typora\Lumen.assets\image-20240123103053265.png)

### Surface Cache资源管理

#### Card Atlas

Lumen通过Lumen.SceneXXX来存储Material Cache和Radiance Cache，**这种数据结构叫Card Atlas**。

Material Cache对应的RDG资源名称为

![image-20240123212629821](Lumen.assets/image-20240123212629821.png)

Radiance Cache对应的资源名称为



![image-20240123212855760](Lumen.assets/image-20240123212855760.png)

#### Card Capture Atlas

需要一提的是，Lumen通过光栅化对Mesh六个面的进行capture后，会把对应的Albedl、Normal、Emissive、Depth分别存储在四张大型的Render Target上，这就把所有的Mesh的Material Attribute都打包到一起了，存储在一个大图集上（Atlas），**这个Atlas 被称为Card Capture Atlas**，这么做可以降低RT交换的损耗。

上面的Card Capture Atlas只是一个临时结构，最终会通过一个叫CopyCardsToSurfaceCache的Pass将Card Capture Atlas的数据Copy到Card Atlas上。

#### Card Atlas的存储

为了保证Card Atlas有足够的空间存储Material Attribute，Lumen中默认每个Card Atlas的大小为4K×4K，共有Depth、Albedo、Opacity、Normal、Emissive五张Atlas。

如果每张贴图32位，共需要320MB显存，这是比较占用空间的，所以Lumen会通过硬件压缩的格式节约资源，

![image-20240123214019388](Lumen.assets/image-20240123214019388.png)

除了Depth之外都用了硬件压缩格式，这是因为需要通过Depth精确还原世界空间的位置以及精确计算Radiance Occlusion，因此不能用有损格式。





## Radiance Cache

Mesh Card不只记录Material（Surface ）的属性，还需要记录光照的Radiance属性，这是为了间接光的复用，我们不可能在相交点发射无数光线去不停的递归计算间接光，所以我们把每一次的光照结果固定下来。

**分帧算法**

第一帧先把直接光记录到Radiance Cache里面

第二帧以第一帧记录了直接光的Mesh Card作为光源，发射射线去Trace，得到的结果当作下一次的光源

每次得到的结果加上上一帧的结果得到Final Lighting，以此递归，相当于把递归算法拆解到每一帧

![image-20240122170702816](C:\Typora\Lumen.assets\image-20240122170702816.png)

如果对每一个Texel都去Trace光线收集Radiance是现实的，哪怕是分帧，每个Texel都去发射光线还是会造成很大能耗，因为间接光材质默认是Diffues的，所以我们以每16×16为一个单位通过八面体映射的技术用Probe记录Radiance，再转换成SH来降低存储，这样可以大幅度加快实时Trace效率，又因为是低频信号，所以结果也很不错。

![image-20240122171805507](C:\Typora\Lumen.assets\image-20240122171805507.png)

![image-20240122171827482](C:\Typora\Lumen.assets\image-20240122171827482.png)

## Sort Mesh Cards

Mesh Cards里面的Lighting Cache需要实时更新，但如果对整个Surface Cache都更新的话是十分耗时的，因此Lumen里面规定，每帧最多不超过 1024 x 1024 个texels更新，因此我们就需要对Mesh Cards进行桶排序，大概只有前面10%的Mesh Cards可以被更新。

![image-20240122172056259](C:\Typora\Lumen.assets\image-20240122172056259.png)

## 



# Scene Dirct Lighting 

# Scene Indirect Lighting（Radiosity）

# Screen Probe Gather





