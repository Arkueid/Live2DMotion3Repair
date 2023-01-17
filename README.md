# Live2d模型moiton3.json文件修复

## 说明

本人在使用 Live2D Cubism SDK for Native 加载从github上下载的模型文件时，产生了类似数组越界的问题，为了解决这个方案查找了多个文档，最终在官方的Spec仓库中找到了关于motion3.json的说明，该说明已放在 "官方说明" 文件夹下，可以自行查看.  

以下是关于上述数组越界问题的解释:  

直接运行官方的SDK for Native加载从网上下载的模型，可能会产生如下错误  
  

![alt 图片](https://github.com/Arkueid/Live2DMotion3Repair/blob/master/resources/Snipaste_2023-01-17_13-33-09.png)  

导致这个断点，可能原因有：
* 释放了一个未被分配的指针
* 数组访问越界
* 其他对未被分配内存的操作  

经过多层debug，最终找到可能导致断点的地方：  

![alt 图片](https://github.com/Arkueid/Live2DMotion3Repair/blob/master/resources/Snipaste_2023-01-17_13-38-45.png)  

把Count参数增大1000，再次运行项目的时候，发现模型可以正常加载

这么粗略的解决肯定是不行的，于是开始关注这几个参数，发现这几个参数对应motion3.json文件中的curveCount，TotalSegmentCount和TotalPointCount等参数。

文件中的这些参数可能有错误，但是这么算这几个参数呢？

经过查找，在网上找到了两条有关这个的说明，结合官方文档和自己实际计算最终得出的计算的方法  

如下是一个motion3.json文件

```json
{
  "Version": 3,
  "Meta": {
    "Duration": 6.833,
    "Fps": 30.0,
    "Loop": true,
    "AreBeziersRestricted": true,
    "CurveCount": 146,
    "TotalSegmentCount": 1115,
    "TotalPointCount": 1447,
    "UserDataCount": 2,
    "TotalUserDataSize": 0
  },
  "Curves": [
    {
      "Target": "Parameter",
      "Id": "Param_Angle_Rotation_3_ArtMesh53",
      "Segments": [
        0,
        8.956,
        2,
        0.333,
        8.956,
        1,
        0.389,
        8.956,
        0.444,
        7.438,
        0.5,
        5.129,
        1,
        0.756,
        -5.489,
        1.011,
        -10.482,
        1.267,
        -10.482,
        0,
        1.767,
        11.301,
        0,
        2.2,
        -17.088,
        0,
        2.6,
        9.684,
        0,
        3.033,
        -10.562,
        0,
        3.267,
        -3.355,
        0,
        3.6,
        -10.369,
        0,
        4.1,
        10.391,
        0,
        4.467,
        -13.114,
        0,
        5.1,
        5.97,
        0,
        5.633,
        -11.428,
        0,
        6.333,
        8.956,
        1,
        6.422,
        8.956,
        6.511,
        7.597,
        6.6,
        5.129,
        1,
        6.678,
        2.97,
        6.755,
        1.038,
        6.833,
        -0.668
      ]
    },
    {
      "Target": "Parameter",
      "Id": "Param_Angle_Rotation_4_ArtMesh53",
      "Segments": [
        0,
        7.009,
        2,
        0.333,
        7.009,
        0,
        0.6,
        7.017,
        0,
        1.3,
        -12.174,
        0,
        1.9,
        7.293,
        0,
        2.367,
        -13.252,
        0,
        2.767,
        8.575,
        0,
        3.167,
        -6.631,
        0,
        3.533,
        -1.717,
        0,
        3.767,
        -10.758,
        0,
        4.233,
        6.945,
        0,
        4.667,
        -11.214,
        0,
        5.1,
        5.668,
        0,
        5.8,
        -7.739,
        1,
        5.978,
        -7.739,
        6.155,
        6.995,
        6.333,
        7.009,
        1,
        6.478,
        7.021,
        6.622,
        7.017,
        6.767,
        7.017,
        1,
        6.789,
        7.017,
        6.811,
        6.937,
        6.833,
        6.788
      ]
    }
  ]
}
```

一个 Curve 就是 Curves 数组中的一个元素  

Curve 中的 Segments 数组解析如下:  
* 一个 point 包含两个浮点数
* 一个 segment 可以等价为一个 identifier [\[1\]] (表示 segment 类型的整数) 与1个或3个 point 构成的集合  

[\[1\]]: #官方的-curve-的说明
* 一个 Segments 数组最开始的两个数代表一个 point
* 于是 Segments 数组 [num1, num2, num3, num4, num5, num6, num7, num8, ...] 等价于如下定义

```python  
[
    Point(num1, num2), # 开头两个数组成一个单独的point，它们之后是一系列segment

    Segment(
        Identifier1(num3),  # 第三个数是 Identifier 表示 Segment 类型
        Point(num4, num5)  # Identifier 之后是1个或3个 point
        ),

    Segment(
        Identifier(num6),
        Point(num7, num8)
    ) 
    ...
]
```  
解析完毕之后就是写代码计算了  

## 官方的 Curve 的说明  

Curves are made up of points and segment identifiers packed into a single flat array.

A point is a sequence of 2 numbers, where *the first number is its timing in seconds*,
hereinafter simply t, and *the second number its value at t*.

A segment identifier is single number with one of the following values.

| Value | Description |
| - | - |
| 0 | Identifier for linear segment. |
| 1 | Identifier for cubic bézier segment. |
| 2 | Identifier for stepped segment |
| 3 | Identifier for inverse-stepped segment. |

*Each curve starts with the first point followed by the segment identifier*.
Therefore, *the first segment identifier is the third number in the flat segments array*.
Segments can be reconstructed by taking the point before the segment identifier and
combining it with the points until the next identifier (or until the end of the array).
A segment identifier is followed by *1 point in case of linear, stepped, and inverse stepped segments*,
that represents the end of the segment, or *3 point in case of bézier segments*, that represent P1, P2, P3.

The t components of bézier control points are proportionally fixed as follows.

```math
P1.t = (P3.t - P0.t) / 3  
P2.t = ((P3.t - P0.t) / 3) * 2
```

Curves can't be empty.


## 配置环境
* Python 3.10及以上

## 第三方库
* 无

## 使用方法

项目中的motion_spec.py是核心工具，下面介绍它的用法  

### 1. 修复单个motion3.json文件  
自己创建一个main.py的文件
然后把motion_spec.py放在和main.py同级的文件夹下
项目结构  
```
project
    |       
    |--motion_spec.py
    |
    |--main.py
    |
    |---Hiyori
        |
        |--motions
           |
           |--Hiyori_01.motion3.json
           |
           ...
```

main.py
```python
from motion_spec import copy_modify_from_motion

motion_path = "Hiyori/motions/Hiyori_01.motion3.json" #模型文件的路径

save_path = "out/motions" #文件的输出文件夹

copy_modify_from_motion(motion_path, save_path)

```
### 2. 批量修改  

```python
# motion3.json文件路径列表

model_dir = "xuefeng_3"  # 模型文件夹路径

out_path = "./out/motions"  # 输出文件夹路径

motionPathList = load_all_motion_path_from_model_dir(model_dir)

for path in motionPathList:
    # path 是motion3.json文件所在路径
    copy_modify_from_motion(path, save_root=out_path)
```  

模型文件夹应该有如下内容  

![alt 图片](https://github.com/Arkueid/Live2DMotion3Repair/blob/master/resources/Snipaste_2023-01-17_14-22-32.png)

## 参考：  
* https://www.bilibili.com/read/cv5533799 (评论区)
* https://github.com/murcherful/Live2D_Displyer (README)
* https://github.com/Live2D/CubismSpecs/blob/master/FileFormats/motion3.json.md (官方文档)