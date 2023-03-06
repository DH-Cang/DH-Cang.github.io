---
layout: post
title:  "Volumetric Cloud"
date:   2023-03-06 16:12:06 +0800
permalink: /volumetric-cloud/
---

## 概述

一个使用Ray Marching方法的体积云算法

* 对云进行建模，使用噪声模拟云的形状
* 使用Ray Marching方法采样噪声纹理，在屏幕空间中渲染体积云的效果

[My Cloud Implementation](https://github.com/JX-Master/oppo_weather_app_luna)


## 云的建模
我们期望得到一个函数，这个函数描述了空间中任意位置的云密度：
```glsl
float sample_cloud_density(float3 position)
```

### 3D噪声
为了描述云的形状，我们首先需要使用3D噪声模拟出类似云的效果，在这里使用了Perlin Noise和Worley Noise两种基础噪声，并使用FBM（Fractal Brownian Motion 分形布朗运动）的方法进行混合。以下我们仅讨论这些噪声的二维形式，三维噪声只需要在采样和插值函数上做出一些改变即可

#### Perlin Noise
Perlin噪声是一种网格噪声，通过对网格点进行插值来表示

<img src="https://raw.githubusercontent.com/DH-Cang/PicResource/master/images/20230306191840.png" alt="perlin noise" style="zoom:50%;"/>

如上图所示，平面被划分为若干个网格，每个网格节点（比如图中0，1的点）存储了一个随机值。对于该网格覆盖范围中任意一点的噪声值，可以被表示为周围四个网格节点中存储随机值的插值，该插值算法不一定是双线性插值，也可以使用别的插值函数。

可以使用一个哈希函数来代表网格节点中存储的随机值，记为<kbd>hash(x, y)</kbd>，有
```glsl
PerlinNoise(x, y) = Interpolate(
    hash(floor(x), floor(y)),
    hash(floor(x)+1, floor(y)),
    hash(floor(x), floor(y)+1),
    hash(floor(x)+1, floor(y)+1)
)
```

#### Worley Noise
Worley噪声同样是一种网格噪声，与Perlin噪声不同的是，它在每个网格内部保存一个随机的特征点，如下图

![worley](https://raw.githubusercontent.com/DH-Cang/PicResource/master/images/20230306204100.png)

对于二维平面上任意一点的噪声值，表示为到最近的特征点的距离

```
WorleyNoise(x, y)
{
    for each adjacent cells:
        min_dis = min(min_dis, dist(current_point, point_in_adjacent_cell));
    return min_dis;
}
```

#### FBM
FBM分型布朗运动，是通过多个不同频率的噪声加权求和做出来的效果，如

```
WorleyFBM = 
0.625 * WorleyNoise(freq=4) + 
0.25 * WorleyNoise(freq=8) + 
0.125 * WorleyNoise(freq=16)
```

### 噪声采样
通过预先计算并保存上述的噪声，我们得到若干个3D纹理，接下来需要利用这些噪声纹理构造采样函数。参照代码中<kbd>sample_cloud_density</kbd>这个函数，我使用了三个系数：<kbd>base_cloud</kbd>用于采样FBM噪声，<kbd>edge_fade_factor</kbd>用于在边界构造渐变效果，<kbd>bound_soften</kbd>用于在云层的上下边界模拟羽化


## 体积云渲染

得到描述云密度的函数后，我们可以通过Ray Marching光线步进的方法，来计算直射光穿过云层的效果

### Ray Marching

![](https://raw.githubusercontent.com/DH-Cang/PicResource/master/images/v2-73f07cbeca84613209c26d1ff2583583_720w.webp)

如上图所示，对于图片中的每个像素，我们构造出ViewRay向量，并在该方向上不断步进，计算直射光的穿透效果
```glsl
float3 view_ray;
float3 start_position;
float step_length;
float total_light_energy = 0;
for(int step = 0; step < MAX_STEP; step++)
{
    float3 current_position = start_position + step * step_length;
    total_light_energy += get_light_energy(current_position);
}

return total_light_energy + BACKGROUND_COLOR;
```

### 光照模型

Ray Marching的计算可以用下图来表示，c表示相机，p表示片元，有一个点光源进行照明（在天光系统中光源则是一个方向固定的直射光）


<img src="https://raw.githubusercontent.com/DH-Cang/PicResource/master/images/image-20221101171652849.png" alt="image-20221101171652849" style="zoom:50%;" />


$$
L_i(c, -v) = \int_{t=0}^{\|p-c\|}LightFunc(c, c-vt) dt
$$

其中LightFunc函数会根据采样点和摄像机的位置，采样云层密度并计算出光照效果

其主要包括两种函数的乘积：

* 朗伯比尔定律（Lambert-Beer law），描述光穿过云层的衰减系数，这里使用了Beer函数和BeerPowder函数
* Henyey-Greenstein Phase Function，描述直射光经过粒子（云中的小水滴）散射后，指向视线方向的概率

$$
CloudThickness(p_1, p_2) = \int_{t=0}^{1} SampleCloudDesity(p1+(p2-p1)*t) dt
$$

```
LightFunc(c, c-vt) = 
SunLight * 
BeerPowder(CloudThickness(distance from (c-vt) to sun_dir)) * 
PhaseFunction * 
Beer(CloudThickness(c, c-vt)) 
```

## 参考资料

《Real-Time Rendering 4th》 Chapter 14

GPU Pro 7:  Real-Time Volumetric Cloudscapes

[Shader Book about noise](https://thebookofshaders.com/11/)

[Unity URP RayMarching 体积云 知乎](https://zhuanlan.zhihu.com/p/440607144)

[ShaderToy Cloud Shape Noise Sampling](https://www.shadertoy.com/view/3dVXDc)
