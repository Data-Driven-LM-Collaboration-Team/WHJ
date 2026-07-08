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

### 自然语言直接生成 STEP 或 CAD 代码存在的问题
1. 语义不稳定：自然语言中的“中心孔”“侧面槽”“圆角边缘”等描述需要被转译为明确的 CAD 特征类型。CGIR 通过 Feature Graph 将模糊语义固化为结构化节点。
2. 空间关系隐含：很多关系并非显式尺寸，而是相对位置、朝向、贯穿、附着、对齐。CGIR 在 Geometry & Constraint Layer 中显式编码这些约束。
3. 建模顺序重要：CAD 中通常先建基体，再做局部加减，最后修饰。CGIR 通过 Construction Plan Layer 强制规定合理的执行拓扑序。
4. STEP/B-rep 过于底层：直接让 LLM 生成 STEP 实体引用图极易出现引用缺失、拓扑不闭合等错误。CGIR 不直接生成 STEP 文本，而是通过 "CGIR → Construction Plan → CAD Kernel 操作 → B-rep → STEP 导出" 的降级链路，利用成熟 CAD 内核（如 OpenCASCADE）的可靠性来规避手写 STEP 的风险。
5. 可验证性不足：如果中间结构不显式表达期望特征，就无法判断生成结果是否符合描述。Validation Expectation Layer 提供了从 CGIR 派生的、可自动检查的验收标准。

所以中间结构应承担“语义解析、几何规划、建模组织、生成约束和验证对齐”的桥梁作用，使自然语言生成 CAD 成为一个可检查、可调试、可逐步落地的工程化流程，而非一次性的黑箱文本生成。

### 中间结构
以 Feature Graph（特征图） 为核心层，向上承接自然语言语义，向下衔接建模操作序列与 STEP/B-rep 底层表示。

| 层级                               | 核心作用                                                                                                                      |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| **Task Semantics Layer**         | 保留原始自然语言描述，提取对象类型、特征标签、空间关系等高层语义。                                                                                         |
| **Part / Body Layer**            | 定义模型中的实体（body/part）、坐标系、单位，区分 base/additive/cutting/reference 等角色，避免多实体语义混乱。                                              |
| **Feature Graph Layer**          | **核心层**。用节点表示 CAD 特征（如 box\_base、through\_hole、fillet、boss），用边表示特征间关系（如 subtract\_from、union\_with、refines、attached\_to）。 |
| **Geometry & Constraint Layer**  | 为每个特征补充精确的几何参数、定位、方向、草图平面、约束（如 concentric、parallel），并显式标记参数来源（explicit / inferred / default / relative）。                  |
| **Construction Plan Layer**      | 将 Feature Graph 转换为可执行的建模步骤序列，规定每步的输入/输出 body、操作类型及前置条件。                                                                  |
| **Target Mapping Layer**         | 描述 CGIR 如何降级到具体后端（如 OpenCASCADE、FreeCAD），定义每个特征对应的 kernel 操作策略及预期的 B-rep 模式。                                              |
| **Validation Expectation Layer** | 定义生成结果的验收标准：实体数量、预期特征是否存在、B-box/体积范围、面类型、STEP 可打开性与 roundtrip 成功性。                                                        |

### 后续如何与经验记忆对接
后续经验记忆将基于 CGIR 的各层结构进行检索、比对和复用
| CGIR 层级                          | 在经验记忆中的作用                                                              |
| -------------------------------- | ---------------------------------------------------------------------- |
| **Task Semantics Layer**         | 作为**自然语言任务相似性检索**的索引。通过对象类型、特征标签和关系描述，检索历史上相似的设计意图描述。                  |
| **Feature Graph Layer**          | 作为**相似 CAD 结构检索**的结构键。通过特征类型组合与特征关系图，匹配历史上成功生成过的模型拓扑。                  |
| **Geometry & Constraint Layer**  | 用于**参数与约束的比对**。当遇到新任务时，可检索历史案例中相似特征的尺寸、位置和约束配置，进行参数复用或推断补全。            |
| **Construction Plan Layer**      | 作为**成功建模经验的复用单元**。若某类 Feature Graph 的构造计划已被验证可行，可直接复用其步骤序列。            |
| **Target Mapping Layer**         | 用于**后端生成策略的复用**。特定特征组合到特定 CAD kernel 的降级策略一旦跑通，可被后续相同或相似特征组合直接调用。      |
| **Validation Expectation Layer** | 用于**经验筛选与质量门禁**。只有满足验证预期的案例（生成成功、STEP 往返成功、模型语义对齐）才允许进入经验库，避免失败噪声污染记忆。 |

