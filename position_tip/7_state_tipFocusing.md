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
