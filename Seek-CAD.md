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

**ground truth**

<img width="256" height="192" alt="image" src="https://github.com/user-attachments/assets/ffd848a7-482f-41a3-851e-077b190f185e" /> <img width="256" height="192" alt="image" src="https://github.com/user-attachments/assets/dc7fc6d2-d97e-4847-976c-e0942c1feb88" /> <img width="256" height="192" alt="image" src="https://github.com/user-attachments/assets/ab473a1b-d7ff-4d2f-8de8-c2cd7efd4025" /> 


**Seek-CAD**

<img width="256" height="192" alt="image" src="https://github.com/user-attachments/assets/2fb645de-8286-492f-8d55-4ba172b14283" /> <img width="256" height="192" alt="image" src="https://github.com/user-attachments/assets/797859c2-5247-4680-ab76-f027b4a5fb0b" /> <img width="256" height="192" alt="image" src="https://github.com/user-attachments/assets/a3fa76e6-6818-4f36-9397-fb442f73c917" />


**Ours**

<img width="256" height="192" alt="image" src="https://github.com/user-attachments/assets/b5d7a59a-9a2a-42c5-8638-34881b9ebcd1" /> <img width="256" height="192" alt="image" src="https://github.com/user-attachments/assets/2ae36a9f-74ee-421f-be5a-2117292756c8" /> <img width="256" height="192" alt="image" src="https://github.com/user-attachments/assets/02b60fa7-9377-4461-92a2-e2ad65733d9d" />




