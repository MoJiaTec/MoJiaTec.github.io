#48 大世界的场景复杂度管理方案
===

- 作者: 天美 tianpeng
- 文章分类: 游戏客户端架构和技术

<br>
&emsp;&emsp;大世界，首当其冲的三个问题：规模、复杂度/性能、渲染，分别对应内容生产、内容承载和内容呈现。本文聚焦如何解决内容承载问题，即场景复杂度管理。在大世界场景里，通常有大面积地形，大规模植被，大量琐碎静态物件等，在相同的硬件平台下，复杂度管理方案很大程度上决定了场景里填充内容的数量和质量。<br>
&emsp;&emsp;本文描述的大世界场景复杂度管理方案，基于控制理论里的负反馈控制系统，分为三个部分：<br>
1、输入部分。包含复杂度降维，复杂度度量，对象评分计算。在引擎和Gameplay层面，根据游戏定制计算因子和权重，统一计算复杂度和评分，传递给控制器模块。<br>
2、控制器/被控对象部分。包含Visibility检测，LOD系统，Scalability系统，动态分辨率等。该部分根据输入和反馈信号，利用多种不同的复杂度控制算法综合调节系统当前负荷。<br>
3、输出/反馈部分。用于实现Adaptive Performance。根据系统负荷能力，系统当前负荷以及系统指定负荷，传递反馈信号到控制器模块。
&emsp;&emsp;整个系统，最终可以达成如下几个目标：<br>
1、运行时根据平台设定，智能控制场景内容的加载卸载、显示隐藏、LOD控制等。<br>
2、根据平台负载能力和当前负荷，更有效的控制运行负荷，获取平滑的fps。<br>
<br>

# 1.概述
&emsp;&emsp;随着硬件平台和游戏技术的不断进步，呈现到玩家手上的游戏品质也在快速提升，尤其是随着手游市场的崛起，近年来呈现了一批品质拔尖的作品。即便这样，依然无法满足玩家对于3A游戏的期待。3A游戏，大部分都是以大世界的形式来表现游戏内容。即便硬件平台的能力在飞速提升，往往也很难满足大世界场景复杂度的爆发式增长。所以如何控制和调节场景复杂度，在很大程度上决定了场景里填充内容的数量和质量。<br>
&emsp;&emsp;下图是一个典型的大世界场景，可以清楚看到，其显著的特征有，视野比较宽，视距比较远，地图比较大，植被比较多，还会有比较多的风格变换，导致绘制内容的种类比较多，资源的使用、变化比简单一些的游戏复杂非常多。
![大世界\label{fig:bigWorld}](bigWorld.jpg)
<br>

&emsp;&emsp;为简化我们的算法模型、以及篇幅限制等原因，文中并不会考虑渲染管线、光照、后处理等一些高级渲染话题，因为这些技术的使用，大多都可以根据硬件平台能力和玩家喜好，通过渲染选项进行静态配置。所以文中关于复杂度和控制算法的讨论，都是基于简化后的基础模型特性进行阐述和讨论。<br>
&emsp;&emsp;场景复杂度，从广义上讲，就是有场景对象的数量和内容细节决定。更细化一点，每个对象的内容包括Mesh，纹理，材质等核心要素。<br>
&emsp;&emsp;控制场景复杂度，最终的目标就是要让游戏平滑的运行，并尽量保持低功耗。要达到这样的目的，就要从场景对象消耗的CPU、GPU、内存、带宽方面入手。即控制显示哪些对象，对象的加载与卸载，显示与隐藏，对象的质量(LOD)等。<br>
&emsp;&emsp;根据之前的项目经验，我们往往会使用Visibility检测算法、LOD策略等，来简单决定场景物件的显示和显示质量，没有一个很好的衡量标准来检测和反馈有效性和准确性。更没有把这些复杂度控制方法整合到一个统一的系统里面，并通过系统的实际运行数据，来准确或者相对准确的来决定我们使用何种控制手段以及如何控制场景复杂度。<br>
&emsp;&emsp;以上这些，就是我们提出大世界的场景复杂度控制方案的出发点。<br>

## 1.1 渲染框架
&emsp;&emsp;从游戏运行层面看，硬件平台的核心资源包括：cpu，gpu，内存、带宽（这里特指soc架构的移动平台）。
&emsp;&emsp;现代硬件平台和图形API，总的趋势是并行渲染。UE在此基础上，渲染框架也充分利用了并行的特点，如下图
![渲染框架\label{fig:Parallel_Rendering.png}](Parallel_Rendering.png)
&emsp;&emsp;这个是Renderer Thread和RHI Thread的交互示意图。整个UE引擎的框架，大致可分为：<br>
game thread,负责游戏逻辑，提交cpu渲染数据。<br>
renderer thread，负责排序、剔除、生成渲染命令。<br>
rhi thread，负责生成gpu渲染数据，提交渲染命令。<br>
gpu thread，负责执行渲染命令。<br>
&emsp;&emsp;根据这四个线程的功能，他们也构成了UE的手游客户端的性能的几个主要部分：CPU逻辑，CPU渲染，CPU提交渲染命令，GPU渲染。除此之外，内存和带宽也是两个重要的性能点。本方案便是立足于优化这些性能点。


## 1.2 系统框架
&emsp;&emsp;在工业自动控制理论里，有两种常用的控制系统模型：正反馈和负反馈。若反馈信号与输入信号极性相同或变化方向同相，则两种信号混合的结果将使放大器的净输入信号大于输出信号，这种反馈叫正反馈。正反馈主要用于信号产生电路。反之，反馈信号与输入信号极性相反或变化方向相反（反相），则叠加的结果将使净输入信号减弱，这种反馈叫负反馈放大电路和自动控制系统通常采用负反馈技术以稳定系统的工作状态。
&emsp;&emsp;很显然，我们的目标是在任意平台任意场景下，达到一个稳定的FPS输出，所以我们需要采用负反馈控制系统模型。一个典型的负反馈控制系统如下图：

![负反馈控制系统\label{fig:ControlSystem}](ControlSystem.png)
<br>
&emsp;&emsp;结合负反馈控制理论和游戏运行目标，我们设计出如下的场景复杂度控制系统：
![场景复杂度控制系统\label{fig:SceneManagement}](SceneManagement.png)
&emsp;&emsp;整个系统分为三个部分：<br>
输入部分：<br>
场景输入模块，主要指的是场景对象的加载和序列化，生成CPU渲染数据等。<br>
系统指标模块，主要指的是根据硬件平台，以及玩家的设置，预先指定的游戏运行时的系统指标，包括FPS,显示特性，CPU、GPU、内存、带宽、电量消耗等数据。<br>
输出部分：<br>
场景呈现模块，主要指渲染模块。<br>
输出检测模块，主要指实际游戏运行时的系统指标，包括FPS,CPU、GPU、内存、带宽、电量消耗等数据。<br>
反馈控制部分：<br>
场景预处理模块，主要指对于场景对象的预处理，用于场景复杂度降维，把全场景复杂度降至当前视野复杂度，主要是指Visibility检测算法。<br>
反馈控制器模块，主要用于根据反馈的系统指标数据和期望的系统指标数据之间的差异，根据一定的策略发送控制指令到复杂度控制模块。<br>
复杂度控制器模块，根据接收到的控制指令，进行相应操作，如调节LOD、加载卸载对象、显示隐藏对象等。<br>


# 2.输入部分
&emsp;&emsp;场景的复杂度，是场景中所有对象复杂度的总和。关于复杂度的定义，根据之前渲染架构一节对于硬件平台的核心资源分析，我们把对于核心资源消耗的因素，统一称作复杂度要素。为了简化模型和算法原型，我们这里关注的场景对象的复杂度，主要由mesh，纹理，材质等核心要素决定。在确定物件的物理复杂度之后，我们还要根据物件的距离(Distance)、屏幕占比(ScreenSize)、重要程度(根据需要人为定义，比如社交属性等)等因素，并根据预先设计的因子权重系数，计算出对象的评分(Ticket)，这个数据将用于后面控制模块的复杂度控制算法中。<br>

## 2.1 对象复杂度评估
&emsp;&emsp;本方案，对于复杂度的评估，基于简化后的模型，主要包含mesh，纹理，材质三个要素。这三个要素在cook特定平台资源的时候可以得到。其中：<br>
Mesh，主要影响加载时长，内存占用，带宽消耗，以及GPU的ALU计算量。<br>
纹理，主要影响带宽。<br>
材质，主要影响GPU消耗。<br>
对于每个多级LOD Mesh，我们都会赋予每个LOD独立的Mesh、纹理和材质，从而达到通过调节LOD控制复杂度的目的。<br>
&emsp;&emsp;另外，由Mesh引起的drawcall, 直接影响到状态切换和渲染调用带来的cpu、gpu消耗。关于drawcall的优化在下面的复杂度控制一节会提及。本方案没有把drawcall作为一个独立的复杂度因子。\[ \begin{pmatrix} a&b\\c&d \end{pmatrix} \quad
\begin{bmatrix} a&b\\c&d \end{bmatrix} \quad
\begin{Bmatrix} a&b\\c&d \end{Bmatrix} \quad
\begin{vmatrix} a&b\\c&d \end{vmatrix} \quad
\begin{Vmatrix} a&b\\c&d \end{Vmatrix} \]

## 2.2 对象评分(Ticket)计算
&emsp;&emsp;在之前的项目优化中，我们通常会在GamePlay层面根据场景对象的社交属性（比如重要性、关注度等）调整对象的LOD、显示或隐藏，在Engine层面又会根据对象的物理属性（比如Distance、ScreenSize）再次调节对象的LOD、显示或隐藏。这种做法因为数据和控制时机的割裂，常常造成很奇怪的bug，或者调节效果不理想。所以，在本方案中，我们会根据预先设定的因子和因子权重，统一计算对象评分。然后存储在一个管理对象评分的全局对象Ticket Manager中，并按大小排序，之后用于复杂度控制器中，智能调节对象LOD，显示隐藏，加载卸载等。<br>
&emsp;&emsp;在Epic关于堡垒之夜的分享中，指出如何根据不同的因子，计算对象的重要等级，这个重要等级和我们之前提出的对象评分类似，下图可以直观的理解这个概念：<br>
![Bucket1\label{fig:Bucket1}](Bucket1.jpg)
&emsp;&emsp;本方案的对象评分计算公式如下：todo $ E=mc^2 $公式编辑。<br>
&emsp;&emsp;其中：<br>

## 2.5 特殊对象的处理
&emsp;&emsp;大世界场景，比较典型的场景物件有大面积地形，大规模植被等。这两类对象有屏幕占比高，实例数量多的特点，是场景复杂度的重要来源。如何处理这两类对象，可以极大的降低场景复杂度，也是我们需要重点关注的内容。

### 2.5.1 Landscape 
#### 2.5.1.1 地形系统与地形低模代理方案<br>
&emsp;&emsp;UE的地形系统，采用的是Geo-Mipmap，他的优点是LOD生成简单，顶点缓冲区、索引缓冲区可以共用等。另外,UE的地形系统，渲染单位是component，并且地表渲染用的是多层layer根据weightmap texture进行混合的方案，为了减少材质复杂度和采样数，UE会为每个component生成一个独立的material instance。基于这些实现和优化，导致了相应的缺点，如component无法合批渲染，使得大地形的DrawCall很高。LOD的区分只和屏幕占比有关，使得无法区分不同地形区域对于地形mesh的细节需求，如山丘和平地有可能使用相同的LOD,要么因为高LOD造成细节丢失，要么因为低LOD造成顶点浪费等。<br>
&emsp;&emsp;基于上述的分析，在使用ue地形系统来构造大世界，我们选择近处使用地形系统来表现细节，远处使用地形低模代理来表现轮廓的方案。这样可以表现大世界的同时，也极大的降低了复杂度，如DrawCall,材质复杂度等。<br>

&emsp;&emsp;UE原生低模代理的生成,只能指定整个sublevel的网格百分比或者某个特定LOD，然而sublevel通常会包含多个component，每个component代表的地貌需要不同的LOD等级。从而导致，这种代理生成方式，不但会影响美术对于地形的迭代速度，也不能保证视觉效果。因此，我们使用基于误差的分割策略。这样既能保证平地使用较少的顶点数，同时也能保证细节较多的地方(如山丘等)保持较好的细节。
![分割策略\label{fig:SpiltMethod}](SpiltMethod.png)

公式如下：
![分割公式\label{fig:SplitFormula}](SplitFormula.png)

#### 2.5.1.2 地形渲染的合批方案<br>
&emsp;&emsp;虽然地形低模代理方案在一定程度上缓解了DrawCall过高的问题，但同时也会增加Mesh和纹理的使用量，本质上是一种空间换时间的方法。所以根据不同硬件平台的特性，我们需要在地形和低模代理直接取一个合适的边界。所以，我们也需要考虑地形系统自身的合批方案，从而降低地形系统自身的DrawCall消耗。根据上面对于UE地形系统的分析，我们可以使用Virtual Texture,把地形渲染时需要的纹理统一通过VT来存储，因此相同LOD的地形component可以合批渲染。
&emsp;&emsp;VT是比较成熟的渲染技术，这里不过多阐述，需要注意的是要进行压缩格式的处理，比如ASTC，否则带宽消耗会比较可观，下图是VT原理示意图：
![vt\label{fig:virtualtexture}](virtualtexture.png)

### 2.5.2 Foliage 
&emsp;&emsp;UE的植被方案采用的是HISM，内部实现采用K-D Tree来管理实例对象，相对于他的基类ISM，他能够支持分簇LOD实例进行渲染，这个特性使得它比较适合一定范围内植被对象的管理和渲染。但是当范围增大，植被实例增加，HISM导致的CPU、GPU消耗都会极具增加。常用的解决方案是增加植被static mesh的LOD区分力度，也无法很好的缓解这个问题，同时还会引起中远景的植被表现力急剧下降。<br>
&emsp;&emsp;下图是hism渲染示意图：
![hism\label{fig:hism}](hism.png)
<br>
#### 2.5.2.1 Imposter方案<br>
&emsp;&emsp;一棵树通常需要几千面组成，一棵树的Impostor只有一个面片，两个三角形组成。Impostor通过对一棵树的多个方向进行拍照，然后根据相机和树的方向，显示对应的纹理图片。Imposter通常会作为一个独立的mesh LOD存在，因为只有一个LOD，所以可以使用ISM存储Imposter植被群，无论在数据结构还是对象复杂度，都比传统的HISM存储的多级LOD Mesh更高效。为了不降低Imposter的表现力，本方案采用的纹理信息和优化点如下:<br>
1、两张纹理：BaseColor、Normal。<br>
BaseColor，R、G、B通道表示BaseColor；Alpha通道：第0-6位：深度；第7位：Mask。<br>
Normal, R、G 通道表示Normal，B通道：AO信息；Alpha通道：第0-6位, 树木的厚度，用来做SSS处理,第7位, 表示树叶和树杆分类。<br>
2、增强立体感。法线进行球面处理：<br>
  OriNormal：捕捉拍到的法线。<br>
  SphereNormal: 球面法线。<br>
  NewNormal：处理后的法线.<br>
  NewNormal = lerp(OriNormal, SphereNormal, lerpParam);<br>
  其中lerpParam参数(0-1)，越小就越靠近原来捕捉的法线，越大就越靠近球面法线。可以通过调lerpParam参数获取想要的效果。<br>

3、SSS效果。烘培Impostor Normal时，多增加一个渲染pass，用来生成树木的厚度图。该厚度信息保存在Normal纹理的Alpha 通道里面。树木的厚度信息用来做SSS的处理：树木的光线透视主要通过厚度来处理，越厚则光线透视越差，相反，越薄则光线投射越强。<br>
  SSS = SSS * Thickness<br>

下图是实际的Imposter渲染效果<br>
![植被Imposter\label{fig:imposter}](imposter.png)
下面是Imposter用到的纹理资源<br>
![植被ImposterDiffuse\label{fig:ImposterDiffuse_s}](ImposterDiffuse_s.png)
![植被ImposterNormal\label{fig:ImposterNormal_s}](ImposterNormal_s.png)
![植被ImposterSSS\label{fig:ImposterSSS_s}](ImposterSSS_s.png)

# 3.输出部分
&emsp;&emsp;输出部分除了将场景呈现给玩家的渲染模块，还有用于检测系统运行时的指标数据，包括FPS,CPU、GPU、内存、带宽、电量消耗等数据。想要精确的获取CPU、GPU和电量消耗，十分依赖与硬件平台的驱动特性，实际上在大部分情况下，我们没有办法简单有效的拿到这些数据。根据我们设计的系统，可以简化问题需求，我们只需要高效并相对准确的获取FPS、CPU和GPU时间、以及内存和带宽数据。UE本身已经实现了一套性能数据检测机制，可以获取上述数据，具体可以参看UE官方网站的Stats System OverView。

# 4.反馈控制部分
&emsp;&emsp;在介绍反馈控制部分的各个模块之前，我们首先明确两个概念：低帧率和卡顿。<br>
&emsp;&emsp;首先低帧率和卡顿是两种完全不同的瓶颈类型，虽然归根到底都是某个函数执行的过慢引起的，但是定位和解决方法并不一样。低帧率瓶颈是需要统计一段时间内CPU把更多的时钟耗费在了哪些函数上，或统计一段时间内各个函数占用的cpu时间百分比，找到百分比高的将其优化，就会使帧率得到整体的提高。卡顿则是在一帧的一次运行内某段代码的运行产生了比平均情况明显的长时间，需要定义这段代码的起始点，分别进行计时，然后在连续的统计数据中找到峰值。简单来说帧率瓶颈是统计平均的CPU占用，而卡顿是找峰值。<br>
&emsp;&emsp;需要指出的是，本文提出的场景复杂度控制系统，目标在于改善和优化低帧率问题，能缓解部分加载引起的卡顿问题。

## 4.1 复杂度控制模块
&emsp;&emsp;在游戏运行时，当场景对象确定之后，复杂度控制的本质就是如何在设定的策略下，在有限的硬件资源条件下尽可能多的显示更多更好的东西，即在性能和画面之间取得一个较好的平衡。常用的方法包括：<br>
选取合适的对象lod。<br>
合理的显示、隐藏。<br>
适当的加载、卸载。<br>
优化drawcall,无论是静态合批、动态合批、HLOD,都是在空间和时间取得平衡。<br>
其他优化手段诸如gpu driven，本质上是增加剔除的精确性、减少剔除消耗、利用indirect command和vt减少渲染时cpu与gpu的交互消耗以及状态切换消耗。<br>
&emsp;&emsp;在我们方案中，我们的目标是智能控制场景内容的加载卸载、显示隐藏、LOD控制。关于drawcall的优化，可以依赖UE自身功能,离线或运行期进行。gpu driven优化，也可与本方案无缝集成，本方案的输出作为gpu driven的输入，gpu driven在特定情况下可以提高裁剪和渲染效率。基于上述分析，我们方案的复杂度控制模块，会接受三个类型的控制指令：调整LOD、显示隐藏对象、加载卸载对象。这三条指令的调整力度依次增强。其中：<br>
调整LOD，选择特定的LOD，直观的表现是调整画面细节。当Game Thread、Renderer Thread、RHI Thread或者GPU Thread, 或者内存、带宽任意一个核心要素出现超负载时，应该优先考虑选择调整LOD。<br>
显示隐藏对象，包括显示隐藏单个场景对象，和调整对象实例群的scalability。当RHI Thread的CPU耗时过少/过多或者GPU Thread的GPU耗时过少/过多告警时，应该考虑显示/隐藏场景对象。<br>
加载卸载对象，这种策略通常用基于内存方面的考虑。在内存富余时，加载对象；内存告警时，卸载对象。<br>

## 4.2 Visibility检测
&emsp;&emsp;Visibility检测属于场景预处理模块，需要说明的是，他属于Runtime阶段的预处理。实际上，我们可以增加离线预处理模块，即离线检测工具，主要用于自动分析场景各区域复杂度，帮助设计人员更有效的设计场景内容。UE已经集成了一定功能的复杂度离线检测工具，用于检测mesh的顶点数、内存占用、纹理、光照贴图、材质等信息。
&emsp;&emsp;UE目前已经集成了多种Visibility检测算法，按消耗排序依次为：<br>
distance culling，frustum culling, pvs，occlusion culling。<br>
&emsp;&emsp;其中oc根据实现方式可以分为hardware oc, hzb oc, software oc。hardware oc利用图形API Query查询对象的可见性，不会增加DrawCall，但是需要回读GPU数据，所以存在延时的问题，导致Pop in等现象，并且在执行效率和裁剪有效性上也存在一定的问题。hzb oc需要hzb的支持，移动平台通常没有这个数据，而且hzb oc的裁剪偏保守，在裁剪有效性上存在问题。所以移动平台通常使用的时software oc，基于算法和simd的优化，可以保证实时性和裁剪有效性。<br>
&emsp;&emsp;下图是我们根据intel nask oc在移动平台优化定制的示意图：
![oc\label{fig:oc}](oc.png)

## 4.3 场景对象加载
&emsp;&emsp;场景对象的加载与卸载，也是一种复杂度控制手段，他会影响到IO、CPU和内存消耗。我们这里重点讲一下加载问题，因为不合理的加载策略，可能导致卡顿，或者显示延迟的现象。通常，我们需要确定加载什么对象以及LOD，进行异步加载的同时还需要限制每帧的加载时长。<br>
&emsp;&emsp;场景对象的加载，有两个重要指标：加载速度、加载的资源量。<br>
加载速度，影响因素包括：数据存储格式，IO速度，同步异步等。其中数据存储格式跟项目密切相关，包括压缩解压算法，资源大小，序列化速度等。<br>
加载的资源量，这个主要是指一次性加载的多个资源，在UE里有一个sublevel的概念，这个是作为一个整体进行加载的，为不同类型、大小、重要性的场景对象进行不同的sublevel划分方式，对于加载的优化十分重要。<br>

&emsp;&emsp;除了上述的性能指标，还有一个加载策略问题，即在什么时候(when)，加载哪些(what)场景对象。为了解决这个问题，我们将场景对象分为：功能类（影响玩法，比如寻路数据、触发器等）、显示类（不影响玩法，比如装饰用的杂草、石块等）、功能+显示类。另外，还有一个基本概念：热区(Area of Interseting)，即玩家感兴趣的范围。对于MMO，角色战斗范围不过几十米之内，大多数情况下，信息获取范围不足百米。而对于FPS来说，枪械有非常远的攻击距离，战斗范围可能会延伸到0.5公里左右。我们可以使用如下的加载策略：<br>
对于功能类对象，如果是影响全局功能，则预先加载；如果是影响局部功能，则在进入设定的影响区域后再进行加载。<br>
对于显示类对象，根据其对画面的贡献(屏幕占比、透明度等)，选在在合适的距离加载对应的LOD数据。<br>

&emsp;&emsp;另外，UE实现了的Mesh Lod Streaming和Texture Streaming机制，利用这个机制，我们可以只加载指定的[RequiredLodLevel ,MaxLodLevel]区间的Lod集合。游戏运行时，合理使用这个机制，可以有效的控制因加载引起的消耗，并且可以加快显示速度。

## 4.4 Bucket分配
&emsp;&emsp;在Epic关于堡垒之夜的分享中，提到一个Bucket的概念，他指的是为不同重要等级的对象预先分配一个LOD段。我们这里也引入Bucket分配策略，用于调整场景对象的LOD。Bucket分配策略需要根据不同的硬件平台能力和当前选取的资源消耗等级进行合理定制。比如性能较差的移动平台，除了自身控制的角色给的Bucket比较高，剩下的角色的都比较低。下图是堡垒之夜用到的一个Bucket分配策略：
![Bucket2\label{fig:Bucket2}](Bucket2.jpg)

## 4.5 反馈控制器模块
&emsp;&emsp;反馈控制器模块，根据反馈的系统指标数据和期望的系统指标数据之间的差异，根据一定的策略发送控制指令到复杂度控制模块，从而达到调节场景复杂度的目的。<br>
### 4.5.1 反馈控制器流程
1、初始化。根据当前硬件平台，初始化targetFPS、Buckets等级。<br>
2、是否可以进行调节（上一次调节是否结束，当前环境是否允许新的调节动作）。否，等待；是，跳到3。<br>
3、根据当前CPU、GPU消耗(时间)，判断PerformanceLevel。如果CPU、GPU均小于Warning阈值，则为Normal;如果CPU、GPU任意一个大于Throttling阈值，则为Throttling；否则为Warning。<br>
4、根据PerfermanLevel设置对应的TargetFPS。<br>
5、判断当前Memory与目标Memory差异，如果大于安全阈值，则发出卸载对象指令。<br>
6、根据PerfermanLevel和目标CPU、GPU、Memory与当前实际数据的差异进行调节。如果PerfermanLevel为Throttling,则发出卸载对象指令；如果PerfermanLevel为Warning，则发出隐藏对象指令或者降低LOD指令；如果PerfermanLevel为Normal，并且内存小于安全阈值且有需要加载的对象，则发出加载对象指令，否则发出显示对象指令或者升高LOD指令。（需要说明的是，因为我们采用了多级Bucket分配方案，所以如果策略允许调节Bucket Level，则先调Bucket Level，再调整对象LOD）。<br>
7、返回2。<br>

### 4.5.2 反馈控制器代码
~~~
enum EPerformanceLevel
{
	Normal,
	Warning,
	Throttling,
}

enum EPerformanceBottleneckType
{
	CPU
	GPU,
	Memory,
};

class FPerformanceController
{
	bool PreferAdjustBucketLevels();
	bool CanAdjustLOD();
	bool CanAdjustActor();
	bool CanAdjustLoad();

	void RaiseLOD();
	void LowerLOD();

	void ActiveActor();
	void DeactiveActor();

	void LoadActor();
	void UnloadActor();
};
~~~

## 4.6 测试数据
todo

# 5.总结
&emsp;&emsp;本文分析了大世界场景的典型特征，并在介绍现代硬件平台的核心资源和渲染架构的基础上，给出了复杂度的定义。结合工业控制中的负反馈控制理论和游戏运行的目标需求，提出了大世界的场景复杂度控制方案，主要原理是根据反馈的系统指标数据和期望的系统指标数据之间的差异，以及预先制定的控制策略，得到相应的控制指令，最终达到智能控制场景内容的加载卸载、显示隐藏、LOD控制的目标以及平滑的游戏运行体验。<br>
&emsp;&emsp;读者可在此基础上，根据项目的需求以及发布平台，合理定制系统指标模块、预处理模块、反馈控制器模块和复杂度控制模块。

# 参考文献
Adaptive Virtual Texture Rendering in Far Cry 4<br>
Terrain in Battlefield 3: A modern, complete and scalable system<br>
UE4制作多人大地型游戏的优化<br>
Fast Terrain Rendering Using Geometrical MipMapping
Real Time 3D Terrain Engines Using C++ And Dx9