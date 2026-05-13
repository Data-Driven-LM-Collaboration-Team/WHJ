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

<img width="1347" height="1168" alt="ChatGPT Image May 13, 2026, 12_55_32 PM" src="https://github.com/user-attachments/assets/469d527b-e407-4510-b34e-75250fb6d6fa" />

