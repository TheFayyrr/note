    case state_fineAdjust:
        Auto_Focus_Slect=fineAdjust(1);
        break;

# fineAdjust(int searchSelect）
这个 `fineAdjust(int searchSelect)` 函数是一个 **自动聚焦的“精调阶段”核心函数**，它根据不同策略，对图像堆栈或扫描深度方向上的清晰度进行评估，并确定一个“最佳焦点位置”。

---

## 🎯 作用总结：

> **在粗调（coarseAdjust）之后，用更精细的方式确定图像最清晰的位置，进而控制显微设备或摄像头移动到该位置，实现自动聚焦（Auto Focus）！**

---

## 📦 核心变量说明：

| 变量名                      | 含义                             |
| ------------------------ | ------------------------------ |
| `searchSelect`           | 选择哪种搜索方法（1：平均；2：斐波那契搜索；3：曲线拟合） |
| `voteMax_`               | 从前面投票系统获得的多个候选峰值信息             |
| `voteEnd_`               | 动态投票决定的最终候选峰值区间                |
| `maxPoseAverage`         | 当前精调算法推断出的“最清晰的 Z 位置”          |
| `cellParamsFirst`        | 控制位移平台/设备的位置指令结构               |
| `Ump_Goto_Position(...)` | 发出移动指令，让设备移动到目标位置              |

---

## 🧠 分支功能详细解释：

### 🧪 case 1：直接平均法（从投票结果中简单平均）

```cpp
for (...) if (voteEnd_ == voteMax_[i].x) {...}
maxPoseAverage /= sizePose;
```

* 遍历所有投票位置中 `x==voteEnd_` 的数据；
* 对这些点的 `y（maxPose）` 求平均；
* 移动到 `home_z - maxPoseAverage*1000` 对应的位置。

✅ **简单、稳定，但不一定最精确**。

---

### 🧠 case 2：斐波那契搜索法（黄金比例搜索峰值）

```cpp
fibonacci(fiboVector);
for (int i = 11; i > 1; i--) {...}
```

* 设定一个范围：`xPose_1 = -180` 到 `xPose_2 = 220`；
* 使用斐波那契比来决定两个搜索点；
* 对两个点分别计算清晰度值（`fineGetSharpness()`）；
* 每轮迭代淘汰一个较差的点；
* 逐步逼近最清晰点（峰值）。

✅ **比平均更智能，效率高，逼近能力强，适合精密系统聚焦**。

---

### 📉 case 3：曲线拟合法（注释掉了）

* 原意是：从堆栈图像中采样清晰度值，对其做曲线拟合（如二次函数）；
* 拟合后直接求函数极大值点作为精调位置。

✅ **理论上最精确，但需要更多计算和数据准备；目前这段代码被注释掉了。**

---

## 🧭 举个例子帮你理解：

假设你有一堆图像，每一张都是某个深度层（Z 轴）拍的。你想找出最清晰的那一层（也就是聚焦点）。

1. 前面 `cvtAdjust()` 已经大致找到了一个可能的区域（粗调）；
2. 现在进入 `fineAdjust()`，你可以选择三种方式：

   * 简单平均：找出大家都说清楚的那一层，直接平均；
   * 黄金比例斐波那契搜索：用数学策略逐步试出最清晰的点；
   * 拟合：用清晰度曲线直接求极值。

最终，这个位置就作为“精调后的对焦点”，设备就会自动移动过去。

---

## ✅ 总结一句话：

> **`fineAdjust()` 是图像自动对焦流程的“精确定位步骤”，通过不同策略（平均、搜索、拟合）判断出最佳聚焦位置，并控制设备移动过去，完成高精度聚焦。**

---

如果你想深入探索：

* `fineGetSharpness(...)` 是如何算清晰度的？
* `Ump_Goto_Position(...)` 是如何控制设备的？
* 为什么采用斐波那契搜索比二分搜索更好？

我可以继续帮你分析 👇 需要吗？

# fineGetSharpness

🧠 1. fineGetSharpness(...) 是如何计算图像清晰度的？
这个函数的功能是：
📸 获取某个位置的图像 → 计算其清晰度值 y（归一化后）→ 用于聚焦判断。

核心流程：
cpp
复制
编辑
sprintf(filename,imgPath,xPose);       // 根据当前焦点位置，构造图像路径
fiboImg = imread(filename);            // 读取对应位置的图像
resize(fiboImg, ..., Size(720,480));   // 统一图像尺寸
sharpnessFunction(..., funSelect, y); // 计算清晰度
y = y / ySize;                         // 清晰度归一化
sharpnessFunction(...) 是关键，它根据 funSelect 选择一种清晰度评价方法，例如：
0：方差法

1：Laplacian 变换

2：Tenengrad

3：Brenner

4：能量（Energy）

5：Normalized Variance

6：细胞计数（Cell Count）

👉 每种方法的目标是衡量图像的锐利程度（如边缘强度、灰度变化等），值越大通常代表越清晰。


# Ump_Goto_Position
⚙️ 2. Ump_Goto_Position(...) 是如何控制设备的？
这个函数是：

调用底层硬件 API（UMP 接口）控制 XYZD 四维位移平台或光学系统移动到目标位置。

关键点解释：
cpp
复制
编辑
ump_goto_position(...)   // 发送目标坐标（x,y,z,d）和移动速度
ump_receive(...)         // 接收平台状态或反馈
ump_get_status(...)      // 获取当前运行状态（如是否忙）
ump_get_positions(...)   // 获取当前实际坐标
等待设备完成移动：
cpp
复制
编辑
while(ump_is_busy_status(...)) {
    ump_receive(...);          // 持续轮询状态
    ump_get_status(...);       // 是否到达
}
✅ 只有当移动完成才返回，确保后续图像采集在对的位置进行。
我们来详细分析两个核心函数：

---

# 🔍 `fineGetSharpness(...)` 分析（图像清晰度计算核心）

这个函数的作用是：

> 📸 读取某个位置的图像，并使用指定的清晰度评价算法计算其**归一化清晰度值 `y`**，用于自动聚焦算法判断当前焦点好坏。

---

## 📌 1. 函数原型与参数

```cpp
void Auto_Focus::fineGetSharpness(int funSelect, int xPose, double &y)
```

| 参数          | 含义                       |
| ----------- | ------------------------ |
| `funSelect` | 清晰度计算方法编号（控制调用哪种算法）      |
| `xPose`     | 当前焦点位置（单位为某种距离单位，μm 或台阶） |
| `y`         | 输出清晰度值，调用后写入计算结果（归一化）    |

---

## 📐 2. 详细流程解析

### ✅ 路程计算（用于统计）

```cpp
static int xPose_last = 0;
static int distance = abs(-200 - xPose);
```

* `xPose_last` 是上一次测量的焦点位置。
* `distance` 是累计移动距离（用于评估耗时或精度统计）。

### ✅ 特殊处理首帧图像（做归一化基准）

```cpp
if (xPose_last == 0) {
    sprintf(filename, imgPath, -180);
    fiboImg = imread(filename);       // 首张图像：基准图
    resize(fiboImg, fiboImg, Size(720, 480));
    sharpnessFunction(fiboImg, outImg, funSelect, y);
    ySize = intSize(y);               // 记录首图清晰度作为基准
}
```

* 从初始图像（例如位置 -180）读取清晰度 `ySize`，用作归一化因子。

### ✅ 主流程：读取图像 → 计算清晰度

```cpp
sprintf(filename, imgPath, xPose);
fiboImg = imread(filename);                // 读取指定位置图像
resize(fiboImg, fiboImg, Size(720, 480));  // 图像缩放统一
sharpnessFunction(fiboImg, outImg, funSelect, y); // 计算清晰度
```

* `sharpnessFunction(...)` 是选择不同清晰度算法的统一接口。

### ✅ 清晰度归一化

```cpp
y = y / ySize;
```

* 降低绝对值波动影响，提高不同图像可比性。

---

## 🧠 `sharpnessFunction(...)` 可能算法（取决于 `funSelect`）

| `funSelect` | 清晰度方法               | 原理简述            |
| ----------- | ------------------- | --------------- |
| 0           | 方差法                 | 灰度方差越大，图像越锐利    |
| 1           | Laplacian           | 二阶导增强边缘，方差度量清晰度 |
| 2           | Tenengrad           | Sobel 梯度强度      |
| 3           | Brenner             | 像素间差平方的总和       |
| 4           | Energy              | 像素能量总和（适用于细节多图） |
| 5           | Normalized Variance | 方差归一化           |
| 6           | Cell Count          | 对细胞边界或数量敏感      |

---

# ⚙️ `Ump_Goto_Position(...)` 分析（精密运动控制核心）

这个函数控制硬件设备移动到绝对位置（x/y/z/d），并同步等待其完成。

---

## 📌 1. 函数原型

```cpp
int Ump_Goto_Position(params_struct *params)
```

| 参数       | 含义                         |
| -------- | -------------------------- |
| `params` | 存储目标坐标、速度、控制句柄、状态等信息的结构体指针 |

---

## 📐 2. 详细流程解析

### ✅ 发送移动指令

```cpp
ret = ump_goto_position(params->handle,
                        params->target_x,
                        params->target_y,
                        params->target_z,
                        params->target_d,
                        params->speed);
```

* 使用底层 API `ump_goto_position()` 发出绝对位置指令；
* 参数是四轴坐标和速度，`handle` 是控制句柄；
* 若返回值 < 0，则说明指令失败，打印错误。

---

### ✅ 查询设备状态

```cpp
ret = ump_receive(params->handle, params->update);  // 接收状态信息
status = ump_get_status(params->handle);           // 获取当前状态码
```

* 接收设备响应，获得状态信息；
* 状态值表示是否移动中、完成或出错。

---

### ✅ 等待设备完成运动（轮询）

```cpp
while (ump_is_busy_status((ump_status)status)) {
    ump_receive(...);
    status = ump_get_status(...);
    return status;
}
```

* `ump_is_busy_status(...)` 判断设备是否仍在运动；
* 若设备还在运动，继续轮询接收状态；
* 当设备状态标记为“空闲”时，表示运动完成，返回。

---

## ✅ 为什么这样做？

* 保证图像采集、聚焦等行为发生在设备真正到达目标位置之后；
* 保证每一步操作的可控性和可靠性；
* 避免拍虚或未聚焦图像。

---

# ✅ 总结对比表

| 功能   | `fineGetSharpness`            | `Ump_Goto_Position` |
| ---- | ----------------------------- | ------------------- |
| 目标   | 计算图像清晰度                       | 控制设备移动至目标位置         |
| 输入   | 图像编号（位置）                      | 坐标值（x/y/z/d）、速度等    |
| 输出   | 清晰度值 `y`                      | 控制是否成功（状态码）         |
| 技术重点 | 图像处理算法                        | 硬件控制 + 状态同步         |
| 依赖模块 | OpenCV、图像路径、sharpnessFunction | UMP API 接口库         |
| 应用场景 | 自动聚焦、图像质量评估                   | 精密移动、图像对位、自动化流程控制   |

---

如果你需要：

* 可视化 sharpness 曲线随位置变化；
* 进一步分析 `sharpnessFunction` 中每种方法的数学原理；
* 或者想移植 `Ump_Goto_Position` 到 Qt 多线程控制框架中；

我可以继续深入帮你做拆解或设计。是否继续？



