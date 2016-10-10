# 社会力模型的 Netlogo 实现

同其他程序，例如 C ++ 或者 Python 相比，Netlogo 的语法十分简单易读；更重要的是，其无需额外的设置。而其缺点，程序运行的效率，以及环境本身的限制带来的影响并不明显。

本文将介绍如何在 Netlogo 环境下，简洁地实现社会力模型的核心部分。您可以在此基础上增加、修改相应参变量以及逻辑，以达到您研究的目的；或者为在其他环境下的开发带来启发。

本文基于我的本科毕业设计的基础部分，用支持公式渲染的 Markdown 软件或者网页阅读风味更佳，: ）

## 1. 什么是社会力模型

社会力模型（Social-Force Model）是 D. Helbing 与 P. Molnár 于1995年提出的，并在之后不断完善的理论模型，其在计算机疏散仿真领域有着广泛应用。

社会力模型的核心是基于运动公式的一组描述个体自身特征同周围环境联系的方程：

$$ F_i = m_i*dv_i/dt = f_i + \sum_{i(\ne j)}f_j + \sum_W f_{iW},\tag1 $$

其中， $f_i$ 所表示的是个体的内心驱动力，其具体为：

$$ f_i = m_i[v_i^0 (t) e_i^0 (t)-v_i (t)]/τ_i,\tag2 $$

对于特定事物及出口的吸引力 $f_0$，其由形如式.3的公式确定：

$$ f_0= A_i*exp[(r_i-d_i)⁄B_i],\tag3 $$

$ \sum_{i(\ne j)}f_j $ 是个体所受来自其他个体的作用力的合力；而来自单个个体的作用力 $F_{ij}$ 可表示为：

$$ F_{ij}= {A_i*exp[(r_{ij}- d_{ij})/B_i ]+ kg(r_{ij}- d_{ij} )} n_{ij}  - \\κg(r_{ij}- d_{ij} )Δv_{ji}^t  t_{ij},\tag4 $$

而来自障碍物的排斥力 $F_{iW}$ 可表示为：

$$ F_iW= {A_i*exp[(r_i- d_{iW})/B_i ]+ kg(r_i- d_{iW} )} n_{iW}  - \\κg(r_i- d_{iW} )(v_i*t_{iW} )t_{iW},\tag5 $$

除此之外，考虑到个体的从众倾向，模型中通过式.6进行修正：

$$ e_i^0 (t)= N[(1-p_i ) e_i+p_i 〈e_j^0 (t)〉_i ],\tag6$$

其中，$e_i^0 (t)$ 代表修正后个体的运动方向。

注: 上述公式及符号的具体含义请参考: D. Helbing et al. _Simulating dynamical features of escape panic_ [J]. Nature, 2000, 407(6803): 487–490.

## 2. 基于 Netlogo 的社会力模型的实现

在 Netlogo 中，可供操作的主体分为`海龟(turtles)`、`瓦片(patches)`、`链(links)`，以及`观察者(observe)`。我们通过将社会力模型，即式.1分为三部分，并借由`海龟(turtles)`、`瓦片(patches)`、`链(links)`分别表示，以在 Netlogo 环境下实现社会力模型。

### 环境的设置

在交互界面的`设置`选项中将`世界`，即场景调整至合适大小，这里将场景设置为$ 27 * 27 $ 大小的网格。注意将`世界在水平方向循环`和`世界在垂直方向循环`的选项取消。

通过`按钮`以及`数据监视框`等选项为程序添加需要的交互项。这里需要将`setup`和`go`按钮，以及调整初始人数（下称 agent）的`滑动条`与计算场景剩余 agent 数量的`数据监视框`添加在交互界面。注意将`go`按钮设置中勾选`循环执行`。

### 程序的编制

在`程序`界面，我们将完成社会力模型仿真程序的剩余部分。

#### 变量的设置与定义

在程序的开头部分，我们将定义不同的种类的海龟和链，并设置全局变量和有关内置变量。

通过`breed [<breeds> <breed>]`以及`directed-link-breed [<link-breeds> <link-breed>]`函数将主体定义为不同种类，这样可以使其遵循不同的命令。

```Netlogo
breed[hitobito hito]	
;; 设置一种名为“人”的海龟
directed-link-breed[in-forces in-force]
;; 设置名为“个体间相互作用力”的有向链
```

通过`globals [var1 ...]`函数定义全局变量，其能由任何主体访问，并允许在程序任何地方使用。

```Netlogo
globals[		;; 定义全局变量
  Ai Bi		 	;; 定义式.3中的参数 A_i 与 B_i
  k kappa		;; 定义式.4及式.5中的参数 k 与 κ
  pre			;; 定义一个中间参数
  num-hibobito 	;; 定义一个初始海龟数量，即人数的参数	
]
```

通过`<breeds>-own`、`<link-breeds>-own`，以及`patches-own`函数，为相应主体赋予不同的属性与参数。同全局变量不同，内置变量对于不同个体可以完全不一样。注意：通过属性，而非坐标或者颜色来区分不同的 patch 将为程序带来极大的便利。

```Netlogo
hitobito-own[	;; 定义“人”的内置参数
  v0			;; 定义初始速度
  v				;; 定义实际速度
  mass			;; 定义质量
  radii			;; 定义半径(社会力模型中通常将 agent 视为圆形粒子)
  tau			;; 定义松弛时间 τ_i
  fidx fidy		;; 定义水平及竖直方向的个体间相互作用力
  fiWdx fiWdy	;; 定义水平及竖直方向的障碍物排斥力
  p_i			;; 定义式.6中的从众系数
]

in-forces-own[	;; 定义“个体间相互作用力”的内置参数
  g				;; 定义重力加速度 g
  dij			;; 定义个体间距离 d_ij
  fij			;; 定义个体间相互作用力 f_ij
  fij-nor		;; 定义 f_ij 的法向分力 
  fij-tan		;; 定义 f_ij 的切向分力
  fijdx fijdy	;; 定义 f_ij 在水平与竖直方向上的分力
  ang			;; 定义 f_ij 即有向链的角度
]

patches-own[	;; 定义 patches 的内部参数
  wall?			;; 定义属性为“墙”的 patch，作为障碍物
  floor?		;; 定义属性为“地板”的 patch
  exit-space?	;; 定义属性为“出口”的 patch
  safe-space?	;; 定义属性为“安全区”的 patch

  d				;; 定义 patch 关于出口距离的参数
  f-exit		;; 定义 patch 关于出口吸引力的函数
  f-patch		;; 定义 patch 对 agent 吸引力的函数
  fiW			;; 定义 patch 关于障碍物排斥力的函数
]
```

为了更好的描述实际的疏散情况，还可以为 agent 添加诸如：性别、年龄、健康程度等参数；为 patch 添加诸如：通过的难易程度、该位置人员拥挤程度等参数。

#### agent 以及场景的设置

这一部分冗长而无趣，请允许我暂时省略。

#### 社会力模型的表示

在这一部分，我们将以在空间中的 floor 区域为例，介绍社会力模型的程序表达。而对于靠近墙壁处，以及出口附近 agent 的运动逻辑，则需要依据相应情况进行设计。

```Netlogo
to evacuate	;; 定义关于疏散的语句

  ask-concurrent hitobito[
	;; 使用并发请求，使 agent 在每一 turn 执行下列命令
	
    let h-ac ((count hitobito)/ninnzuu) * h
    create-in-forces-to other hitobito in-radius 0.7
	;; 命令 agent 生成一条指向质心距离 0.7 单位内其他 agent 有向链
	;; 对于 f_ij 而言，其是一个随着距离增加而迅速下降的函数
	;; 选取合适的距离作为生成 f_ij 可以相应减少部分计算量
	
    if ([float?] of patch-here = true)
    ;; 判断所处 patch 属性是否为 float
    [let p min-one-of neighbors [f-patch]
    ;; 将相邻 Moore 领域 patch 的最小值赋予临时变量 p
    ;; 这将用以指示 agent 的运动方向
    
    if [f-patch] of p < (f-patch)[
    ;; 判断所处 patch 的吸引力函数是否大于 p
    ;; （在本程序中对于吸引力函数，
    ;; 其数值越小表示对于 agent 的吸引力越大）
      set fidx ((mass * (sin towards p * v0 - sin heading * v))/tau)
      set fidy ((mass * (cos towards p * v0 - cos heading * v))/tau)
    ]
   	;; 分别设置内心驱动力的水平竖直分量

    let w min-one-of neighbors [fiW]
    if [fiW] of w < fiW [
      set fiWdx ([fiW] of w * sin towards w)
      set fiWdy ([fiW] of w * cos towards w)
    ]
    ;; 同理，设置并分解障碍物排斥力

    let sum-fijdx (sum [fijdx] of my-in-in-forces)
    let sum-fijdy (sum [fijdy] of my-in-in-forces)
    ;; 同理，合成并分解个体间相互作用力

    let fdx (fidx + fiWdx + sum-fijdx)
    let fdy (fidy + fiWdy + sum-fijdy)
    ;; 将所有力在水平竖直方向合成

    let vdx (v * sin heading) + (fdx * n-tick / mass)
    let vdy (v * cos heading) + (fdy * n-tick / mass)
	;; 得到水平竖直方向的分速度，其中 n-tick 代表 1/ frames
	;; 这是为了使运动更加平滑
	
    let cor (atan (vdx + 0.01) (vdy + 0.01))
    ;; 得到速度的角度；在分速度基础上加上一个小数
    ;; 目的在于避免求余切时出错

    let a-h (mean [heading] of hitobito in-cone 2 120)
    ;; 用临时变量 a-h，即 average-heading
    ;; 储存 agent 视角范围内其他个体的平均角度

    set heading ((1 - p_i) * cor - a-h * p_i)
    ;; 设置最终的运动方向
    
    …
  
  ]
end
```

### 仿真的运行与数据处理

Netlogo 贴心的提供了`行为空间`这一工具，便于多次、重复的进行仿真实验。并可以通过 .csv 格式输出实验数据，便于对结果的分析与处理。您可以在`工具`-`行为空间`中找到它。



以上です
