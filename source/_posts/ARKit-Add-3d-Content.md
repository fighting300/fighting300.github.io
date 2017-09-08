---
title: ARKit上手指南 01 添加3D物体
date: 2017-09-08 12:56:22
tags:
   -ARKit
---

ARKit是2017年6月6日，苹果发布的iOS11系统所新增的框架,它能够帮助我们以最简单快捷的方式实现AR技术功能。而且ARKit框架对基于3D场景(SceneKit)和2D场景(SpriteKit)的增强现实都提供了支持。

#### 开发环境  
1. Xcode版本： Xcode9以上   
Xcode 9 Beta：[https://developer.apple.com/download/](https://developer.apple.com/download/)
下载最新的beta版本就可以了，不过Xcode需要Mac 10.12.4及以上的版本。  
2. iOS系统： iOS11以上
3. iOS设备： A9/A10处理器的iOS设备，即iPhone6s、iPad2017及以上的设备

#### 创建项目

首先打开Xcode，选择ARKit模板，如下所示：  

![AR项目创建](http://ojca2gwha.bkt.clouddn.com/ARKit_1Project.png)

之后，填写完项目信息后，选择Content Technology为SceneKit，当然也可以选择SpriteKit，不过在3D空间中就不是那么立体了。
开发语言选择Swift，Swift天然亲和ARKit，很多网上的Demo都是用Swift写的，这样也方便移植和借鉴。   
<!--more-->
然后连接你的测试设备并运行，app就可以运行了。该模版APP会在实施摄像头镜头中展示一架飞机的3D模型。如下图所示：  

![3D飞机](http://ojca2gwha.bkt.clouddn.com/AR-Plane.jpg)

实际项目中，你也可以不使用该模版来创建项目，直接引入相关库也可以进行开发。

在项目中可以看到`viewWillAppear `方法中已经初始化了ARWorldTrackingConfiguration实例。

```
override func viewWillAppear(_ animated: Bool) {
      super.viewWillAppear(animated)  

      // Create a session configuration
      let configuration = ARWorldTrackingConfiguration()

      // Run the view's session
      sceneView.session.run(configuration)
}
```
#### 放置3D物体

SceneKit有一些基础类，SCNScene是所有3D内容的容器，可以在其中添加多个3D物体。
要向scene中添加内容，要创建SCNGeometry，然后将其包装为SCNNode并添加到SCNScene中。

首先注释掉`let scene = SCNScene(named: "art.scnassets/ship.scn")! sceneView.scene = scene`，然后添加代码如下：

```
override func viewDidLoad() {
	super.viewDidLoad()
	// 存放所有3D几何体的容器
	let scene = SCNScene()

	// 想要绘制的 3D 立方体
	let boxGeometry = SCNBox(width: 0.1, height: 0.1, length: 0.1, chamferRadius: 0.0)

	// 将几何体包装为node以便添加到scene
	let boxNode = SCNNode(geometry: boxGeometry)

	// 把box放在摄像头正前方
	boxNode.position = SCNVector3Make(0, 0, -0.5)

	// rootNode是一个特殊的node，它是所有node的起始点
	scene.rootNode.addChildNode(boxNode)

	// 将 scene 赋给 view
	sceneView.scene = scene
}

```

现在运行该项目，就会看到有3D立方体悬浮在空中，并且全方位无死角。

此外还可以增加一些调试信息
```
	// ARKit统计信息例如fps等
	sceneView.showsStatistics = YES;

	sceneView.debugOptions = [ARSCNDebugOptions.showFeaturePoints];
	// 调整摄像头属性 当前摄像头有效直径在10m范围内
	if let camera = sceneView.pointOfView?.camera {
		camera.wantsHDR = true
       camera.wantsExposureAdaptation = true
       camera.exposureOffset = -1
       camera.minimumExposure = -1
       camera.zFar = 10
	}

```


之前简单体验了ARKit的功能，下面简单介绍ARKit的工作原理：   

#### ARKit工作原理
在ARKit中，创建虚拟3D模型其实可以分为两个步骤：   
1. 相机捕捉现实世界图像--由ARKit实现   
2. 在图像中显示虚拟3D模型/2D模型--由SceneKit/SpriteKit实现     

ARKit中ARSCNView用于显示3D虚拟AR的视图，它的作用是管理一个ARSession，一个ARSCNView实例默认持有一个ARSession。
在一个完整的AR体验中，ARKit框架只负责将真实世界画面转变为一个3D场景，这一个转变的过程主要分为两个环节：由ARCamera负责捕捉摄像头画面，由ARSession负责搭建3D场景，而将虚拟物体显示在3D场景中则是由SceneKit框架来完成，每个虚拟物体都是一个节点SCNNode，每个节点构成一个场景SCNScene。  
ARCamera只负责捕捉图像，不参与数据的处理。它属于3D场景中的一个环节，每一个3D Scene都会有一个Camera，它决定了我们看物体的视野。
下图是ARKit与SceneKit的框架关系图:   

![ARKit class结构](http://ojca2gwha.bkt.clouddn.com/ARKit-class.png)

ARSessionConfiguration的主要目的就是负责追踪相机在3D世界中的位置以及一些特征场景的捕捉（例如平面捕捉），这个类本身比较简单却作用巨大。ARSessionConfiguration是一个父类，为了更好的看到增强现实的效果，苹果官方建议我们使用它的子类ARWorldTrackingSessionConfiguration，该类只支持A9芯片之后的机型，也就是iPhone6s之后的机型。

当ARWorldTrackingSessionConfiguration计算出相机在3D世界中的位置时，它本身并不持有这个位置数据，而是将其计算出的位置数据交给ARSession去管理，而相机的位置数据对应的类就是ARFrame。ARSession类一个属性叫做currentFrame，维护的就是ARFrame这个对象。  

![ARFrame](http://ojca2gwha.bkt.clouddn.com/AR-Session.png)

ARKit的完整运行流程可以参考下图：
1. ARSCNView加载场景SCNScene  
2. SCNScene启动ARCamera开始捕捉图像  
3. ARSCNView开始将SCNScene的场景数据交给ARSession  
4. ARSession通过管理ARSessionConfiguration实现场景的追踪并且返回一个ARFrame(添加3D物体模型时计算3D模型相对于相机的真实矩阵位置时需要使用)
5. 给ARSCNView的SCNScene添加一个子节点(SCNNode)

![ARKit工作流程](http://ojca2gwha.bkt.clouddn.com/AR-flow.png)

本文将会使用ARKit创建一个简单的app，结束时就可以在AR世界里放置3D物体，并且可以用iOS设备绕着它移动。虽然这是一个非常简单的app，我们会在之后的文章中继续为其编写更多功能，包括平面检测、3D物理效果等其他东西。   
