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
