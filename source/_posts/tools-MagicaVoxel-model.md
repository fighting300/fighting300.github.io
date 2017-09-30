---
title: 快速创建ARKit模型-使用MagicaVoxel创建3D模型    
date: 2017-09-30 13:55:31
tags: [tools, ARKit]
categories: Tool
---

iOS APP尤其是游戏中，还有新发布的ARKit中，2D/3D模型需要花费大量的时间来创建，然而我们开发者开没有那么长时间。  
最近发现一款及其简单的模型创建工具-MagicaVoxel，虽然他只能创建`体素(Voxel)`模型，但是生成的模型可以在ARKit中使用，而且它还是免费的。
下面我们介绍下该软件。

![Voxel模型](http://ojca2gwha.bkt.clouddn.com/MagicaVoxel-example4.png)

#### 开始使用  
可能你会疑惑体素是什么意思，其实体素类似与像素，像素英文为Pixel，是图片pictures(pix)与元素element(el)俩词组合而来，而体素Voxel则由体积Volumes(vox)与元素element(el)组成。尤其是当你玩过`我的世界`的话，很快就能理解，里面的模型其实由一个个体素构成，它是一种极其简单、快速创建3D模型的方式。而MagicaVoxel就是一种创建Voxel模型的工具。

你可以从该地址下载MagicaVoxel： https://ephtracy.github.io，然后解压它。  
当你在Mac上运行APP发现黑屏时，你需要把`MagicaVoxel-mac.app`文件移出当前目录，再移动回来，就可以解决黑屏问题。  

<!--more-->

打开APP后界面如下图所示：  

![MagicaVoxel Model界面](http://ojca2gwha.bkt.clouddn.com/MagicaVoxel-layout.png)

界面被划分为8个基本区域，分别为Palette(调色板)、Brush(画笔)、ViewOptions(视图)、Editor(编辑)、FileOptions(文件)、ExportOptions(格式导出)、镜头、View(视图)区域。当你切换到Render模式时，界面会发生变化，新加了Light(光影)、Matter(材质区域)。

![MagicaVoxel Render界面](http://ojca2gwha.bkt.clouddn.com/MagicaVoxel-renderlayout.png)

当前MagicVoxel中可以创建126*3立方体的最大画布，在右上侧的size中可以修改画布大小。  

> 在画布中按住右键可以旋转视角，空格+右键可以拖动画布，滚轮缩放画布。
> Mac中根据右键设置进行相应的操作，一般为两指轻触触控板。

#### 开始创作  

打开APP后，会首先看到一个立方体，你需要先清空该内容，在Tool中选择Zreo，从0开始，然后改变画布大小，设置View区域size大小为最大即126.

开始前介绍下Brush工具，其总共有六种模式，分别为单个模型(V)、面模型(F)、区域(B)、直线模型(L)、圆模型(C)、选择文件已有的模型(P),如名字所代表的意思，在相应模式下可以创建相应的模型。

![Brush工具](http://ojca2gwha.bkt.clouddn.com/MagicaVoxel-brush.png)

下面开始简单的创作MagicxVoxel的图标。

**1.颜色选取**
在颜色面板选择最后一个，可以用默认配置的颜色面板、或者创建你自己的颜色面板，在下方的HSV中选择想要的颜色。
调色板中0、1是默认的不同颜色的排布，2是灰阶色的排布，3为自定义色板。左侧E/G可以修改视图里辅助线和地面的颜色。   

**2.人物搭建**
在Brush中选择B模式，用Attach搭建模型，并且可以在Mirror镜像中选择X轴，这样你在左边画的东西会复制到右边。  
创建的过程就像垒积木，最终垒成你想要的样子，注意每个体素都有6个面，可以在相应的面上垒其他体素。

**3.Paint上色**

之后，你可以关掉Mirror模式，用Paint工具，给相应的体素上色，使人物更具像素感觉。  

**4.添加层级**

此时，magic模型看上去只有一面，你可以选择F模型，在头部点击，同颜色的相邻色块就会添加一层厚度。

![Model](http://ojca2gwha.bkt.clouddn.com/MagicaVoxel-model.png)

**5.渲染**

完成上述步骤后，就可以进入Render模式。
在左边的Light工具栏中，根据需求调整shadow(阴影)、I-sun(光照)、I-sky(天空)、Fog(雾气)等参数。
在Shape工具栏中，可以选择不同的形状，包括Lego、MC、RG、RE、Sphr、Cyli这6种性形状。
同理，还可以在右侧的Matter工具中(可以选择All(全部)或Sel(选中))，更改Diffuse、Metal、Glass、Emission等不同的材质配置。

下图就是渲染完成后的模型效果，做的还是比较像的～

![Magic渲染效果图](http://ojca2gwha.bkt.clouddn.com/MagicaVoxel-render.png)  

在最新的版本MagicaVoxel-0.98中还增加了动画帧的效果，可以给每帧增减一些体素，然后制作gif，比较酷炫。

#### 生成ARKit可用模型  

上述步骤后，我们已经创建了MagicaVoxel原生格式.vox的模型，但是ARKit使用的SceneKit并不支持该格式，幸运的是该软件可以将模型输出为常见的`.obj`格式，该格式被很多其他3D创作工具所支持。

**导出.obj文件**  
在Export区域，选择`.obj`就可以导出所需的文件，该文件会保存在Application/MagicaVoxel/export目录下。

![](导出obj模型文件)[]

这时会输出3个文件，分别为：
1.monu10.mtl(材料库文件，包含颜色定义、纹理和反射贴图)
2.monu10.obj(.obj文件，包含体素模型的几何体信息)
3.monu10.png(漫反射纹理贴图，包含了模型中所用到的颜色)
现在可以将导出的模型，导入到Xcode中，然后转换为SceneKit场景文件了。

**转换为.scn文件**

在你项目的**art.scnassets**目录下，添加上述导出的.obj文件和.png文件。
选中.obj然后在Xcode菜单中选择Editor下的`Convert to SceneKit file format (.scn)`，covert即可替换原来的.obj版本。
这样monu10.obj文件已经转换为合适的monu10.scn文件了。

![](转换scn模型文件)[]  
之后就可以在材料检查器、节点检查器和物理检查器中调整相应的Lighting mode(光照模型)、Diffuse、物理形体等参数了。

好了，以上这样就完成了一个新的模型，这样比别的工具是快了很多，只是不能够创建接近真实的模型。如有问题，请评论或微博私信～

再上一张酷炫的图...

![City Voxel](http://ojca2gwha.bkt.clouddn.com/MagicaVoxel-example1.jpg)

参考：  
MagicVoxel作者的微博: [gltracy的微博](http://weibo.com/gltracy)  
资源/视频教程地址：[resource](https://ephtracy.github.io/index.html?page=mv_resource#)
