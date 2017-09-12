---
title: ARKit 上手指南 02 平面检测
date: 2017-09-12 20:53:07
tags:
   ARKit
---

上一小节中，我们在ARKit虚拟世界中添加了3D物体，接下来我们会使用ARKit检测现实世界的平面。这样我们才能在平面上放置物体以及进行其他操作。

#### 检测平面  

如果要在ARKit空间里检测水平面，需要设置ARSessionConfiguration的planeDetection属性为ARPlaneDetectionHorizontal。设置完后，ARSceneView的delegate方法会收到回调。

```
func renderer(_ renderer: SCNSceneRenderer, didAdd node: SCNNode, for anchor: ARAnchor) {
    DispatchQueue.main.async {
        if let planeAnchor = anchor as? ARPlaneAnchor {
            self.addPlane(node: node, anchor: planeAnchor)
        }
    }
 }
```

每次ARKit认为检测到平面时都会调用该方法，并且返回一个ARAnchor对象，包含当前检测平面锚点，其真正类型是ARPlaneAnchor，拥有extent(范围)、center(中心)等属性信息。
```
open class ARPlaneAnchor : ARAnchor {
    // The alignment of the plane.
    open var alignment: ARPlaneAnchor.Alignment { get }

    // The center of the plane in the anchor’s coordinate space.
    open var center: vector_float3 { get }

	 // The extent of the plane in the anchor’s coordinate space.
    open var extent: vector_float3 { get }
}
```

#### 渲染平面

根据第一步得到的锚点信息，我们可以在ARKit的空间中绘制一个3D平面。创建一个Plance类，继承自SCNNode来管理需要渲染的平面，然后创建SCNGeometry子类SCNPlane的对象，可以用该对象创建平面对应的SCNNode。在构造方法中创建平面并调整其大小：

```
init(_ anchor: ARPlaneAnchor) {
	...
	// 用ARPlaneAnchor的尺寸来创建3D平面几何体
	let plane = SCNPlane(width: CGFloat(anchor.extent.x - 0.05), height: CGFloat(anchor.extent.z - 0.05))
	let material = SCNMaterial()
	material.colorBufferWriteMask = []
	material.isDoubleSided = true
	plane.materials = [material]

	planeNode = SCNNode()
	planeNode!.geometry = plane
	// SceneKit 里的平面默认是垂直的，所以需要旋转90度来匹配 ARKit 中的平面
	planeNode!.transform = SCNMatrix4MakeRotation(-Float.pi / 2.0, 1, 0, 0)
	// 将平面plane移动到ARKit对应的位置
	planeNode!.position = SCNVector3Make(anchor.center.x, -0.01, anchor.center.z)
}
```

然后在ARSCNViewDelegate的回调方法中，每次扫描到新的ARAnchor时都可以创建新的平面。

```
    func renderer(_ renderer: SCNSceneRenderer, didAdd node: SCNNode, for anchor: ARAnchor) {
        DispatchQueue.main.async {
            if let planeAnchor = anchor as? ARPlaneAnchor {
              let plane = Plane(anchor)
		        node.addChildNode(plane)
            }
        }
    }

```

#### 更新平面

上述创建渲染的平面只会保持创建时的大小，然后现实世界的平面可能比创建初始化的大很多，若需要更新平面的大小，需要更新平面的extent(范围)值。

可以从ARSCNViewDelegate的代理方法中获取到更新的信息：

```
    func renderer(_ renderer: SCNSceneRenderer, didUpdate node: SCNNode, for anchor: ARAnchor) {
        DispatchQueue.main.async {
            if let planeAnchor = anchor as? ARPlaneAnchor {
	            plane.update(anchor)
            }
        }
    }

```

然后在Plane类的update方法里，更新plane的宽高：
```
	func update(_ anchor: ARPlaneAnchor) {
		plane.width = CGFloat(anchor.extent.x - 0.05)
		plane.height = CGFloat(anchor.extent.z - 0.05)
		// 更新位置
		plane.position = SCNVector3Make(anchor.center.x, -0.01, anchor.center.z)
	}

```
