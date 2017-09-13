---
title: ARKit-基于Face的AR体验
date: 2017-09-13 11:58:19
tags:
  ARKit
---

Apple昨晚发布了强大但是死贵无比的iPhoneX，惊呆了我们。然后只能默默去刷了下API，发现ARKit新添了基于Face的API。
之前的版本中前置摄像头是不支持AKKit模块的，幸而新发布的版本中添加了对Face识别的支持，然而新的API需要原前置深感摄像头(TrueDepth Camera)的支持，所以看来还得入手一台iPhone X呐。

![iPhoneX](http://ojca2gwha.bkt.clouddn.com/ARKit-iPhoneX.jpg)  

#### 新增API
首先新增了ARFaceTrackingConfiguration，以用于追踪用户脸部的运动和表情，并在界面中渲染虚拟内容。此外你还可以设置lightEstimationEnabled为true，来启用脸部扫描并提供预估的定向/环境光。

其次还新增了ARFaceAnchor，这两个新API与之前的ARWorldTrackingConfiguration和ARPlaneAnchor类似，只是基于平面的检测改为了基于Face的检测，恶趣味下Apple会不会继续扩展他的检测API。。。当使用ARFaceTrackingConfiguration时，ARSession会检测用户面部并且用ARFaceAnchor保存面部信息，包括位置、方向、拓扑结构以及表情特征点。

ARFaceAnchor的transform属性描述了面部在世界坐标系中的所在位置和方向，该世界坐标系可以在ARSessionConfiguration的wordlAlignment中设置。可以使用该transform在ARScene检测到的面部上添加虚拟物体。坐标系如下图所示：

![ARKit Face coordinate](http://ojca2gwha.bkt.clouddn.com/ARKit-face-Coordinate.png)

该坐标系为右手坐标系，x轴只想观察者的右边(即face自己的左边)，y轴指向上边(与face绑定)，z轴指向了观察者。

ARFaceAnchor的blendShapes属性提供了当前面部表情的高级数据，通过一系列表示面部特征的系数来描述面部表情。你可以使用这些系数跟随用户的面部表情来做2D/3D动画，

#### 开始面部跟踪

和ARKit的其他用法一样，面部追踪需要ARSessionConfiguration并运行一个ARSession来渲染相机屏幕，这部分可以参考[之前分享的文章](httpbai.ducom)，流程一致，只是这部分需要使用到新加的API。
如前面所说，面部检测只支持配置了原前置深感摄像头(TrueDepth Camera)的iPhone，所以首先需要判断当前设备是否支持ARFaceTrackingConfiguration。

```    
    // 如果设备支持的判断通过 然后配置Session Configuration
    guard ARFaceTrackingConfiguration.isSupported else { return }
    let configuration = ARFaceTrackingConfiguration()
    configuration.isLightEstimationEnabled = true
    session.run(configuration, options: [.resetTracking, .removeExistingAnchors])
```

#### 跟踪脸部位置和方向  
当面部跟踪启用后，ARKit会自动的添加ARFaceAnchor到ARSession中，包括其位置和方向。  

> 注意ARKit目前只能追踪单个用户的面部，如果多个用户同时出现在镜头中，则取最大或者最清晰的可识别面部

你可以在代理方法`renderer:didAddNode:forAnchor:`(ARSCNViewDelegate协议)中给一个face锚点添加相应3D内容。他会跟随用户的面部动作做相应的位置和方向调整。  

```
func renderer(_ renderer: SCNSceneRenderer, didAdd node: SCNNode, for anchor: ARAnchor) {
    let device = sceneView.device!
    let maskGeometry = ARSCNFaceGeometry(device: device)!
    let content = SCNNode(geometry:maskGeometry)
    node.addChildNode(content)
}
```

##### 使用Face Geometry来模拟用户面部

ARKit提供了粗粒度的3D网状几何体，来匹配用户的面部大小、形态、拓扑结构以及当前面部表情，当然ARKit还提供了ARSCNFaceGeometry类来快捷的初始化该网格。你可以在用户的皮肤上通过给网状模型添加材质来绘制虚拟的刺青或者面具等。

```
let device = sceneView.device!
let maskGeometry = ARSCNFaceGeometry(device: device)!
```

为了让屏幕中面部模型匹配用户的脸部，甚至跟随用户的眨眼、说话或者更多的表情动作，需要在代理方法中更新该面部模型的数据。
```
func renderer(_ renderer: SCNSceneRenderer, didUpdate node: SCNNode, for anchor: ARAnchor) {
    guard let faceAnchor = anchor as? ARFaceAnchor else { return }

    let faceGeometry = content.geometry as! ARSCNFaceGeometry
    faceGeometry.update(from: anchor.geometry)
}

```

#####  在面部呈现3D内容

ARKit提供的面部网格的另一个用途是在场景中创建遮挡几何体。遮挡几何体是不会呈现任何可见内容（允许相机图像显示）的3D模型，但会阻碍相机对场景中其他虚拟内容的视图。这种技术创造出真实面对与虚拟对象交互的错觉，即使脸部是2D摄像机图像，虚拟内容是渲染的3D对象。例如，如果您将遮挡几何图形和虚拟眼镜放置在用户的脸部上，则脸部可能会眼镜框架遮挡。
要创建面部的遮挡几何，首先创建一个ARSCNFaceGeometry对象，如前面的例子所示。但是，不要使用可见的外观来配置该对象的SceneKit材质，而是在渲染期间将材质设置为渲染深度而不是颜色：

```
    let geometry = SCNGeometry()
    geometry.firstMaterial!.colorBufferWriteMask = []
    occlusionNode = SCNNode(geometry: geometry)
    occlusionNode.renderingOrder = -1
```

##### 让你的形象动起来  
要获取当前用户的面部表情，需要在代理方法renderer:didUpdateNode:forAnchor:中从ARFaceAnchor的blendShapes中读取数据 。

```
func renderer(_ renderer: SCNSceneRenderer, didUpdate node: SCNNode, for anchor: ARAnchor) {
    blendShapes = faceAnchor.blendShapes
}
```
然后，查看该字典中的键值对以计算模型的动画参数。该字典中有52个独特的ARBlendShapeLocation系数。你可以使用尽可能少的或很多的必要条件来创建想要的艺术效果。在此示例中，RobotHead类执行此计算，映射ARBlendShapeLocationEyeBlinkLeft和ARBlendShapeLocationEyeBlinkRight参数到机器人眼睛的scale参数，用ARBlendShapeLocationJawOpen参数以计算机器人上颌所在的位置。
```
var blendShapes: [ARFaceAnchor.BlendShapeLocation: Any] = [:] {
    didSet {
        guard let eyeBlinkLeft = blendShapes[.eyeBlinkLeft] as? Float,
            let eyeBlinkRight = blendShapes[.eyeBlinkRight] as? Float,
            let jawOpen = blendShapes[.jawOpen] as? Float
            else { return }
        eyeLeftNode.scale.z = 1 - eyeBlinkLeft
        eyeRightNode.scale.z = 1 - eyeBlinkRight
        jawNode.position.y = originalJawY - jawHeight * jawOpen
    }
}
```

先简单介绍到这儿，真正要看到效果还得公司的土豪们入手iPhone X之后啦，不过基本用法很简单，维持了ARKit易用的特性。

![致敬Jobs](http://ojca2gwha.bkt.clouddn.com/ARKit-Jobs.jpg)

###### 参考文档
1. https://developer.apple.com/documentation/arkit/arfacetrackingconfiguration?language=objc
2. https://developer.apple.com/documentation/arkit/creating_face_based_ar_experiences?language=objc
