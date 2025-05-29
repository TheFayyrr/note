    case state_tipFocusing://针尖聚焦
        //planeSelection=tipFocusing(300,3,6);//范围、投票阈值、拟合点个数
        planeSelection=tipFocusing(100,3,6);//范围、投票阈值、拟合点个数 2micro
//        planeSelection=tipFocusing(100,3,6);//范围、投票阈值、拟合点个数
        break;

# 

你贴出的 `Pose_Plane::tipFocusing()` 函数是一个 **“自动针尖聚焦”流程的核心实现**。我来帮你详细分析其：

---

## 🧠 **功能目标：**

自动控制探针（针尖）在 Z 轴方向移动并采图，计算图像清晰度（锐度），通过拟合清晰度曲线找到最佳焦点位置（focus peak），从而实现 **针尖自动聚焦**。

---

## 📌 **函数定义与输入：**

```cpp
int8_t Pose_Plane::tipFocusing(int range, int threshold, int fitNumber)
```

* `range`：投票范围（移动范围判断是否聚焦成功）
* `threshold`：投票阈值（拟合的清晰度曲线阈值）
* `fitNumber`：曲线拟合点数量（最近多少个点参与聚焦估计）

---

## 📋 **完整工作流程拆解：**

---

### ✅ **步骤 1：图像读取与保存**

```cpp
if(cameraOpen)
{
    nikoncamera->grabImage(imageRead);
}
else
{
    sprintf(filename,"./tipfocus/4/focusimg%d.jpg",numFlag);
    test_img = imread(filename);
    if (test_img.empty()) return state_idle; // 视频/图像结束
}
```

* 支持两种模式：

  * 摄像头实时采集（如 Nikon 相机）
  * 从视频帧 / 图片序列读取（脱机调试）
* 图像保存到本地路径（如 `./tipfocus/focusimgX.jpg`）

---

### ✅ **步骤 2：位置记录与初始化**

```cpp
if(numFlag == 0)
{
    timer.start(); // 开始计时
    emit sendposrec(1,4); // 发出开始记录信号

    // 记录当前位置
    Ump_Select_Dev(&line2DParams);
    Ump_Read_Position(&line2DParams);
    tipClearPosition = line2DParams;
}
```

* 第一次运行时：

  * 开始计时
  * 发出“位置记录”信号
  * 记录初始探针位置（用于后续回归）

---

### ✅ **步骤 3：图像区域提取与清晰度评价**

```cpp
imgFocus = test_img; // 可裁剪区域
sharpnessFunction(imgFocus, outImg, 4, focusMeasure); // 计算清晰度（4=方差）
```

* 清晰度评价使用 `sharpnessFunction(...)`

  * 方法参数 `4` 代表“图像方差”（Variance）作为锐度指标
  * 结果存入 `focusMeasure`

---

### ✅ **步骤 4：保存结果，拟合准备**

```cpp
curve.push(QPointF(stepMicro * (numFlag - 1), focusMeasure)); // 构建曲线点
if(curve.size() > fitNumber) curve.pop(); // 限制点数
```

* 每个图像帧对应一个清晰度点 `(z偏移, 清晰度)`
* 保存到 `curve` 队列中，便于后续曲线拟合

---

### ✅ **步骤 5：判断是否完成聚焦（拟合结果判断）**

```cpp
if(curve.size() == fitNumber)
{
    if(numFlag > 65)
    {
        // 估计最大清晰度位置 voteEnd_tip
        tipClearPosition.home_z += voteEnd_tip * 1000;

        // 控制探针移动到最佳焦点位置
        Ump_Goto_Position(&line2DParams);
        Sleep(500);

        emit sendposrec(0,4); // 发出停止记录信号
        handle.close();
        return state_idle;
    }
}
```

* 达到一定拟合点后（如 `numFlag > 65`）：

  * 拟合清晰度曲线，获取最大值点 `voteEnd_tip`
  * 探针回到该清晰度最高的位置（最佳焦点）
  * 发出“结束记录”信号
  * 回到 `state_idle`，表示聚焦完成

---

### ✅ **步骤 6：未完成聚焦时继续移动**

```cpp
line2DParams.target_z = line2DParams.home_z + numFlag * stepMicro * 1000;
Ump_Goto_Position(&line2DParams);
Sleep(500);
numFlag++;
return state_tipFocusing;
```

* 探针每次向下移动 `stepMicro`（=2μm）
* 继续采图、评价、拟合，直到拟合满足结束条件

---

## 📊 总结工作流程图：

```text
初始化
 ↓
图像采集（相机/图片序列）
 ↓
图像裁剪 + 清晰度计算
 ↓
加入曲线点（Z偏移 vs 清晰度）
 ↓
曲线长度未满 → 下一帧（向下移动 stepMicro μm）
 ↓
曲线长度足够 → 判断拟合结果是否满足（voteEnd_tip）
 ↓
若满足 → 回到清晰度峰值点（Z轴），停止
 否则继续采图移动
```

---

## 🧰 实现的功能核心：

| 功能模块     | 描述                                            |
| -------- | --------------------------------------------- |
| 图像采集     | 支持相机实时采图或视频/图像序列离线处理                          |
| 清晰度计算    | 使用图像锐度函数（如方差）计算图像清晰度                          |
| 数据记录     | 保存图像与清晰度值，输出文本和图像                             |
| 曲线拟合判断   | 判断清晰度曲线是否达到聚焦峰值，估计最清晰位置                       |
| 探针自动移动控制 | 控制 Z 轴电机自动移动针尖并聚焦到最优位置                        |
| 状态转换     | 聚焦完成返回 `state_idle`，未完成继续 `state_tipFocusing` |

---

## ✅ 结论

`tipFocusing()` 是一个**全自动、数据驱动的聚焦流程**，利用锐度计算 + Z轴扫描 + 曲线拟合完成针尖聚焦。其关键设计理念如下：

* **逐步移动+逐帧采图**
* **锐度评估 → 聚焦点检测**
* **最终返回最佳 Z 坐标点**

---

如果你还想继续实现：

* 绘图查看 `focusMeasure` 曲线拟合效果
* 支持多种锐度指标切换（如 Tenengrad、Brenner）
* 优化曲线拟合精度和退出条件判断

我都可以帮你实现 👇。需要的话，告诉我你的下一步目标！


# `Pose_Plane::tipFocusing()` 函数主要输出了三类数据，分别是：

---

## 📤 一、**图像输出**

### ✅ 1. 采集图像帧保存：

```cpp
imwrite(filename, test_img);
```

* 路径格式：`./tipfocus/focusimg%d.jpg`
* 内容：原始采集图像（来自相机或图像序列）
* 保存频率：每次采图都保存一次，编号为 `numFlag`

---

### ✅ 2. 聚焦区域图像（ROI）保存：

```cpp
imwrite(filename, imgFocus);
```

* 路径格式：`./tipfocus/focusroi%d.jpg`
* 内容：用于聚焦分析的图像区域（ROI），目前是整图
* 可修改为裁剪的区域：`imgFocus = test_img(Rect(...))`

---

## 📤 二、**清晰度评价数据输出**

### ✅ 3. 清晰度日志写入：

```cpp
write_ << text << endl;
```

* 路径：`./txt/tipfocus.txt`

* 内容格式：

  ```
  numFlag:0;focusmeasure:1532.827393; Time:23.000000
  numFlag:1;focusmeasure:1625.273438; Time:45.000000
  ...
  ```

* 每一行保存一次图像采集对应的：

  * `numFlag`：当前帧编号
  * `focusmeasure`：当前图像的清晰度评价值
  * `Time`：从开始采集起经过的毫秒数（使用 `QElapsedTimer`）

---

## 📤 三、**视频输出（可选）**

### ✅ 4. 创建视频记录：

```cpp
static VideoWriter videoCreate("tipfocus.avi", ...);
```

* 视频文件：`tipfocus.avi`
* 帧率：25fps
* 图像尺寸：2048×542（灰度图设定）
* 当前代码中并未 `videoCreate.write(...)` 添加帧（你可以添加）
* 可用于后期查看聚焦全过程

---

## 🔁 可输出的数据汇总表：

| 输出内容       | 文件路径                        | 数据类型      | 用途         |
| ---------- | --------------------------- | --------- | ---------- |
| 每帧采集图像     | `./tipfocus/focusimg%d.jpg` | 图像文件（jpg） | 保存原始图像用于调试 |
| 清晰度计算区域图   | `./tipfocus/focusroi%d.jpg` | 图像文件（jpg） | 用于锐度函数分析   |
| 清晰度日志（含时间） | `./txt/tipfocus.txt`        | 文本文件      | 记录聚焦曲线数据   |
| 聚焦过程视频（可选） | `tipfocus.avi`              | 视频文件      | 回放聚焦过程（灰度） |

---

## ✅ 建议（可选拓展）：

你可以考虑增加以下数据保存：

* ✅ **焦点拟合结果保存**（voteEnd\_tip、拟合曲线图）
* ✅ **锐度曲线可视化输出（保存成PNG）**
* ✅ **最终焦点图像裁剪并保存（FinalROI.jpg）**

如果你需要我帮你 **画出锐度变化曲线** 或将所有输出打包导出成一个报告（如PDF），也可以继续告诉我。

# 输出的分析
你提供的这些“输出信息”来自多个部分，可以分为三类，分别对应程序的不同运行阶段与模块。以下是详细解读：

---

### ✅ **1. 你自己的程序输出（`qDebug()`、`cout`、`emit`、变量值）**

这些是你在 `tipFocusing()` 函数中主动打印的信息：

```
tip Focusing is running !
position has been   sent
numFlag: 0;focusmeasure: 35.1203
numFlag: 1;focusmeasure: 35.2232
numFlag: 2;focusmeasure: 35.2924
numFlag: 3;focusmeasure: 35.3172
```

解释：

* `tip Focusing is running !`：每次函数开始运行时通过 `qDebug()` 打印，说明聚焦过程启动了。
* `position has been sent`：`emit sendposrec(1,4)` 之后打印的说明信号发出成功。
* `numFlag: x; focusmeasure: y`：表示当前帧编号（或聚焦步骤数）和对应图像的清晰度（通过方差方法计算）。

---

### ⚠️ **2. OpenCV 插件加载信息**

这类是 **OpenCV 自动打印的日志信息**，不是你程序主动输出的，但它们不是错误，更多的是提示插件加载的状态：

```
[ INFO:2@133.089] global backend_plugin.cpp:383 ... Found 2 plugin(s) for GSTREAMER
...
[ INFO:2@133.091] global plugin_loader.impl.hpp:67 ... load opencv_videoio_msmf480_64d.dll => OK
...
[ INFO:2@133.533] global cap_msmf.cpp:1027 ... Using D3D11 video acceleration on GPU device: NVIDIA GeForce RTX 3080 Ti
```

这些说明：

* OpenCV 在初始化 `VideoCapture` 和 `VideoWriter` 时，会尝试加载插件（如 GStreamer、Intel MFX、MSMF 等）。
* 有些 DLL 加载失败（可能你没安装 GStreamer 或 Intel MFX 支持），但最终成功加载了 MSMF（微软媒体基础）插件，并在 RTX 3080 Ti 上启用了硬件加速。
* 不影响程序运行，**可以忽略**这些提示。

---

### ❌ **3. 控制系统通信失败（读位置失败）**

这部分是你的运动控制或机械臂相关模块出现的问题：

```
read positions failed - Invalid dev id
read positions failed - Timeout occured
```

这说明：

* `Ump_Read_Position(&line2DParams)` 调用失败。
* 可能原因：

  * `manipulationSelection` 的设备 ID 错误。
  * 下位机/控制器通信中断或超时。
  * 控制接口未初始化或未连接。

💡 **建议你排查：**

* 是否 `actOpen` 为真（设备连接成功）。
* `manipulationSelection` 的值是否设置正确。
* 是否控制软件/硬件启动正常。

---

### 🔁 **tipFocusing 的流程逻辑简要回顾**

每次 `tipFocusing()` 被调用：

1. 读取一张图像（来自相机或图片序列）。
2. 计算该图像的清晰度（`sharpnessFunction()`）。
3. 显示清晰度并存入曲线队列。
4. 如果到达聚焦步数上限（如 65），执行拟合、运动到焦点。
5. 否则通过 `Ump_Goto_Position()` 控制位移平台向下微调（如步长 2 微米），进入下一次循环。

---

### ✅ 总结

| 输出类型                                       | 来源            | 说明                     |
| ------------------------------------------ | ------------- | ---------------------- |
| `tip Focusing is running !`、`numFlag: ...` | 你的程序逻辑        | 显示聚焦进度和图像清晰度           |
| `[INFO:...]`                               | OpenCV 插件加载日志 | 可忽略，非错误                |
| `read positions failed`                    | 控制器通信失败       | 检查下位机连接、设备 ID 设置、初始化状态 |

---

如你希望进一步排查 `read positions failed` 的具体原因，可以贴出 `Ump_Read_Position()` 或 `Ump_Select_Dev()` 的实现或日志。我也可以协助你添加更详细的错误处理逻辑。


