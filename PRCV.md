## GeoViewCAD: Image-to-CAD Generation via Three-view Structured SVG Recovery

### 任务定义
核心任务：给定一张包含规则几何物体的 RGB 图像，自动生成可编辑的参数化 CAD 模型。

关键难点：原始图像（像素）与 CAD 模型（参数化建模指令序列）之间存在巨大的表示鸿沟，直接映射困难且不稳定。

解决思路：引入三视图结构化 SVG 作为关键中间表示，将难题拆解为两步：
1. 图像 → 前视/俯视/右视三视图结构化 SVG
2. 三视图 SVG → 参数化 CAD 指令序列

### 关键模块
1. Geometry-aware Image Encoder
* 提取 CAD-oriented 几何特征（轮廓、孔槽、长直线、圆弧、对称性）
* 可使用 RGB、边缘图、mask、透视/方向线索
2. Three-view SVG Proposal Decoder
* 从单图像特征生成初始三视图 SVG（front/top/right）
可采用：
* Template-assisted retrieval（检索最相似模板并微调）
* Direct view-specific decoding（primitive-level预测）
3. View-wise SVG Structure Graph Refinement
* 构建每个视图的结构图（节点：primitive、endpoint、closed profile；边：连接、重合、平行、同心等）
* 执行修复操作：端点对齐、path 合并、重复去除、闭环构建、hole/slot tagging
4. Cross-view Consistency Reasoning
* 建立跨视图关系，保证三视图共同描述同一个 3D CAD object
典型约束：
* 共享尺寸（width/height/depth）
* 孔/槽对应
* 外轮廓和对称轴一致性
5. Three-view SVG Tokenization
* 将三视图 SVG 转换为 CAD decoder 可读的 primitive token sequence
* 保留视图分隔、闭环、孔/槽、跨视图对应信息
6. Drawing2CAD-style CAD Sequence Decoder
* 基于三视图 SVG tokens 生成 CAD command sequence
* 双解码器结构：命令类型 decoder + 参数 decoder
* 可执行 CAD 验证（sketch closure、extrude validity）
