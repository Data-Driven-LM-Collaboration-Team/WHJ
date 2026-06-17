## GeoViewCAD: Image-to-CAD Generation via Three-view Structured SVG Recovery

### 研究问题
CAD的重要性：CAD模型与简单的渲染3D形状有着本质区别；它本质上是一个可执行的几何程序。这些程序包含构建历史、草图、封闭轮廓、约束、拉伸、切除以及特定的操作参数。

论文研究“图像转CAD”（Image-to-CAD）生成问题，旨在将单个物体图像转换为结构化、参数化且面向CAD的表示形式。现已有方法通过三视图 SVG 作为中间结构作为后续 CAD 生成的输入，但从单张含有瑕疵或噪点的图片到结构化的 SVG，得到的 SVG 往往效果不佳。

### 挑战
* 单张图片到结构化SVG
  图像包含纹理、光照、透视和遮挡，而三视图 SVG 需要描绘出每个视角的具体拓扑信息。如何让模型能够精准地去根据有瑕疵的图像推理出完整的三视图。

<img width="392" height="128" alt="image" src="https://github.com/user-attachments/assets/07c838fb-dd6c-4d42-a450-a9d9c5cb89f9" />
