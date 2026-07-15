<img width="1636" height="425" alt="image" src="https://github.com/user-attachments/assets/43bf7712-f2eb-4e11-908e-2d0c6854d28f" />


<img width="2560" height="1249" alt="image" src="https://github.com/user-attachments/assets/de83752e-5315-4d19-b076-2ae466fdf0a6" />

CAD Description: 一个具有恒定厚度的三维实体，呈心形轮廓，具有两个凸出的叶状部分和中央的凹陷，从顶部看形成对称形状。

<img width="2560" height="1249" alt="image" src="https://github.com/user-attachments/assets/8ba3424e-954d-4b64-85c1-b6c95281a830" />

### SSR code
```
from math import cos, sin, radians
# Create main cylinder body  
sk0 = Sketch(plane={"normal": [0,0,1], "origin": [0,0,0], "x": [1,0,0]})  
p0 = Profile()  
p0.addLoop(  
    Loop().moveTo(0,0).circle(50)  # Radius=50mm  
)  
sk0.addProfile(p0)  
cylinder = Extrude(sk0, distance=(0, 20))  # Height=20mm  

# Create Pac-Man cutout  
sk1 = Sketch(plane={"normal": [0,0,1], "origin": [0,0,20], "x": [1,0,0]})  
p1 = Profile()  
p1.addLoop(                  
    Loop()  
    .moveTo(0,0)  
    .lineTo(50,0)  # Start radial line  
    .threePointArc([25,43.3], [0,0])  # 120° arc (Pac-Man mouth) # 此时画笔已经在 (0,0)
    .lineTo(0,0)  # Close wedge                                  # 画一条从 (0,0) 到 (0,0) 的直线
)                                                                # pythonocc中画一条长度为0的边不合法，渲染失败抛出错误
sk1.addProfile(p1)  
pacman_cut = Extrude(sk1, distance=(0, -5))  # Cut depth=5mm  

# Create square indentations  
sk2 = Sketch(plane={"normal": [0,0,1], "origin": [0,0,20], "x": [1,0,0]})  
for i, angle in enumerate([0, 120, 240]):  
    x = 45 * cos(radians(angle))  # Offset from center  
    y = 45 * sin(radians(angle))  
    p2 = Profile()  
    p2.addLoop(  
        Loop()  
        .moveTo(x-2.5, y-2.5)  
        .lineTo(x+2.5, y-2.5)  
        .lineTo(x+2.5, y+2.5)  
        .lineTo(x-2.5, y+2.5)  
        .lineTo(x-2.5, y-2.5)  # 5x5mm square  
    )  
    sk2.addProfile(p2)  
indentations = Extrude(sk2, distance=(0, -3))  # Indentation depth=3mm  

# Combine operations  
final_shape = cylinder.cut(pacman_cut).cut(indentations)  
```

## Exp_SSR

**Seek-CAD RAG Text2SSR 示例**
<img width="1646" height="654" alt="image" src="https://github.com/user-attachments/assets/92c78517-a3f3-4727-9038-a9043978b68a" />


**Exp经验记忆库每条记忆在Text2SSR的基础上额外写入**
```
memory_text              用于 exp_rag 向量嵌入和 sparse（精确关键词）匹配
memory_summary           注入 prompt 的简短可复用经验
intent_tags              从描述解析出的 CAD 意图标签
geometry_terms           几何关键词和同义词
operation_counts         SSR 代码中的 Sketch/Profile/Extrude/cut/union 等统计
construction_patterns    例如 extrude_based、subtractive_cut、sketch_arc_profile
construction_recipe      可复用的构造步骤摘要
risk_tags                例如 plane_orientation、arc_closure_validity、topology_reference_validity
```


**Exp card**
从若干个Seek-CAD生成案例中总结经验，为每个案例生成一条经验。


**检索实现**
使用bge-m3 hybrid。Baseline的 case 检索索引原始自然语言描述 + SSR Code; Exp_SSR 的 case 检索默认索引经验记忆库的 `memory_text`，其中包含任务语义、intent tags、geometry terms、construction patterns、operation counts、construction recipe 和 risk tags。检索结果仍返回原始 SSR code，供 prompt 作为可执行风格示例。

## STEP 文件生成与记忆库构建思路

**STEP 文件**：内部描述了三维模型的几何信息、拓扑关系以及产品结构信息。一个 STEP 文件通常由 `HEADER` 和 `DATA` 两部分组成：`HEADER` 记录文件来源、单位、标准版本等元信息；`DATA` 则通过大量带编号的实体描述模型本身（点、方向、曲线、曲面、边、环、面、壳体和实体等）。这些实体之间通过 `#id` 形式相互引用，从而构成一个完整的几何-拓扑结构。基于这一特点，STEP 文件具备一定的“可读性”和“结构化文本”特征，因此可以被大语言模型解析和理解。

<img width="2560" height="1249" alt="image" src="https://github.com/user-attachments/assets/bb887cb5-78bc-46b6-bcf0-13a776657ab7" />

**description: A 90-degree pipe elbow connector.**

当前已经尝试让 LLM 根据自然语言描述直接生成 STEP 文件。虽然模型往往难以一次性生成完全正确、可直接打开并编辑的 STEP 模型文件，但 LLM 能够在后续多轮交互中读取生成文件的内容，分析其中可能存在的实体引用错误、结构缺失、格式不完整或拓扑不一致问题，并逐步进行修复。经过若干轮检查与修改后，模型可以输出能够在 SolidWorks 等 CAD 软件中打开的 STEP 文件。这说明 LLM 已经具备一定的 STEP 文件语义理解能力和错误修复能力。

**目的**：引入记忆库机制，去提升 STEP 文件生成的成功率和模型准确性。构建记忆库的前提是设计一种结构化的中间格式。该中间格式可以记录模型的基础形体、局部特征、空间关系、关键 STEP 实体模式，例如孔、槽、倒角、圆角、凸台、壳体等常见 CAD 特征如何在 STEP 文件中被表达。通过这种方式，系统可以将成功生成的案例沉淀为可复用的经验记忆，在面对新的自然语言描述时检索相似结构或相似特征，辅助 LLM 更稳定地生成 STEP 文件。

**失败案例及其修复过程**：对于无法打开、无法渲染或结构错误的 STEP 文件，可以记录其错误类型、错误位置、失败原因、修复策略以及修复后的有效结果，形成失败案例修复记忆库。后续当系统生成类似错误的 CAD 模型时，可以根据错误签名或 STEP 结构特征检索历史修复经验，指导 LLM 自动修复失败模型。因此，记忆库的作用不仅体现在生成阶段，也体现在验证与修复阶段。

## 中间结构构成

### 自然语言直接生成 STEP 存在的问题
1. 语义不够精确：自然语言中的“中心孔”“外边圆角”“侧面槽”等说法，需要先转成明确的 CAD feature、方向、位置和作用对象。
2. STEP 结构过于底层：STEP/B-rep 由点、边、环、面、壳体、实体等大量引用关系组成，直接生成文本很容易出现引用缺失、面不闭合、方向错误或拓扑非法。
3. 建模顺序重要：CAD 中通常先建基体，再做局部特征加减，最后做修饰。CGIR 通过 Construction Plan Layer 强制规定合理的执行拓扑序。
4. 失败难以定位：如果直接生成 STEP，失败时很难判断问题来自语义理解、参数推断、建模顺序、后端操作，还是 STEP 导出。
5. 结果需要语义验证：文件能打开不代表语义正确，还需要验证孔、圆角、bbox、surface 类型等是否符合预期。

所以中间结构应承担“语义解析、几何规划、建模组织、生成约束和验证对齐”的桥梁作用，使自然语言生成 CAD 成为一个可检查、可调试、可逐步落地的工程化流程，而非一次性的黑箱文本生成。

### 中间结构
以 Feature Graph（特征图） 为核心层，向上承接自然语言语义，向下衔接建模操作序列与 STEP/B-rep 底层表示。

| 层级                               | 核心作用                                                                                                                      |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **Task Semantics Layer**         | 保存原文、对象类型、已用特征和未覆盖需求。                                                                                         |
| **Part / Body Layer**            | 定义 mm 单位、坐标系和一个或多个 solid body。                                              |
| **Feature Graph Layer**          | 定义特征及其先后依赖。                                                                                     |
| **Geometry & Constraint Layer**  | 定义参数、局部坐标系、闭合轮廓和边/面选择器。                  |

当前执行器支持 22 类特征：5 类 base（box、plate、cylinder、ring、profile extrusion），6 类切除（贯通孔、盲孔、slot、pocket、groove、notch），4 类加料（boss、rib、pillar、flange），3 类修饰（fillet、chamfer、shell）和 4 类 pattern（线性、圆周、镜像、显式实例）。孔、轮廓和加料的位置必须通过已存在实体的局部 frame 表达；pattern 只能复制已执行的同一 body 的加料或切除特征。

能力边界：

- 不支持 sweep、loft、revolve、自由曲面、B-spline 曲面或任意草图约束。
- 可导出多个互不相连的 solid 为 compound STEP，但不支持装配层级、装配约束或跨 body boolean。
- shell 只适用于一个可解析的 opening face；复杂 boolean 之后的任意面/边拓扑命名和通用薄壳并不稳定。
- PMI/GD&T、制造语义和未给出的尺寸、位置、数量均不能由 CGIR 自动补成“正确答案”。
           
### 最新实验

- CGIR 校验通过：7/10；STEP 成功导出：6/10。
- 失败：2 条 Schema 违规、1 条同一 body 有 5 个 base feature、1 条生成空实体。
- 9/10 条用满 thinking budget；没有输出截断。因此下一步优先补结构约束和修复，不应先加大 token 上限。
- 71 个参数来自默认策略、11 个被模型写为 inferred；原文没有显式尺寸。渲染出的绝对尺度只是默认值，不能当成文本中的真实尺寸。

### 文本与结果的人工核对

下表只判断描述中可直接观察到的主特征；它不是 ground-truth CAD 评测。

| 样本 | 对照结果 | 判断 |
| --- | --- | --- |
| 003，十字挤出体与圆角 | 十字主轮廓和圆角外观可见。 | 主体吻合；圆角目标边被宽泛选择|
| 004，梯形棱柱与沿长度的 V 槽 | 梯形主体正确；CGIR 中 V_GROOVE_PROFILE 实际是矩形。 | **不吻合**：V 槽被替换为矩形槽 |
| 005，圆端棱柱与两端圆孔 | 两个孔存在；圆端轮廓写成八边形折线。 | 部分吻合：以折角近似圆端 |
| 006，贯通矩形空腔与三个等距孔 | 矩形体、中央切口、三孔均生成。 | 主特征存在；腔体方向、孔距和阵列方向均由默认值决定 |
| 007，三角棱柱与中部三角切除 | 三角柱与顶部三角切口可见。 | 主体吻合 |
| 009，倒角、内腔、四个角部六角凹坑 | 倒角、内腔和六角凹坑可见。 | **不吻合**：CGIR 只有 3 个六角 pocket（F3–F5），文本要求 4 个。 |

<img width="1280" height="540" alt="image" src="https://github.com/user-attachments/assets/f32ae40e-9e94-4e84-8e8b-a52652f6934c" />
