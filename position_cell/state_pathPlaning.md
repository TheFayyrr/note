

    case state_pathPlaning:
        Auto_Focus_Slect=pathPlaning();
        break;

        你提的问题非常好，我们来**逐步解释这个 `pathPlaning()` 函数的作用、逻辑流程，以及 `return state_pathPlaning` 是怎么运作的**。

---
# pathPlanning

## ✅ 一、这个 `pathPlaning()` 函数在干嘛？

### 🌟它的功能简述：

**执行自动路径规划**，让聚焦过程可以**按最优顺序访问细胞位置（barycenter）**，而不是乱序移动，从而提升自动处理效率。

---

### 🔍 详细分步逻辑解析：

```cpp
int8_t Auto_Focus::pathPlaning()
```

* 返回值是 `int8_t` 类型，代表返回一个“状态值”（比如继续干别的事情）。

---

### 🧠 1. 初始化变量

```cpp
static int pCount = 0;       // 静态变量：调用 pathPlaning 函数多少次了
int8_t returnV = 0;
static Mat drawing, cellImg; // 静态图像缓存
```

---

### 🧠 2. 首次调用时：抓取图像、计算细胞质心

```cpp
if(pCount == 0)
{
    cellImg = getCellImg();                 // 拍照获取细胞图像
    barycentersGet(drawing, cellImg);      // 得到每个细胞的质心坐标，画在 drawing 上
    barycentersWrite();                    // 把质心坐标写入文件
    GetDist(barycenterP, barycenterP.size()); // 计算质心间的距离矩阵
    Init(barycenterP.size());              // 初始化路径算法用的结构
}
pCount++;
```

---

### 🧠 3. 使用外部 TSP 求解器（LKH）得到最优访问路径

```cpp
QProcess pro;
pro.execute("../MicroSystem/LKH-3.exe", QStringList() << "../MicroSystem/Cells.par");
pro.kill();
```

* 启动 `LKH-3.exe` 外部路径规划器，读取配置 `Cells.par`，输出路径到 `Best.txt`

---

### 🧠 4. 读取最优路径结果 Best.txt

```cpp
QFile fl("Best.txt");
...
while(fl.atEnd() == false) {
    int cNum = ...; // 从文本中读取每个城市编号（对应质心）
    bestPath.citys.push_back(cNum);
    bestPathPoint.push_back(barycenterP[cNum - 1]); // 转为坐标点
}
```

---

### 🧠 5. 把路径画出来，保存图像

```cpp
for (...) {
    circle(drawing, barycenterP[iNum], 4, Scalar(0, 0, 0), 1);
    line(drawing, barycenterP[iNum], barycenterP[iNum_], Scalar(255, 0, 0), 1);
}
imwrite("cell_8path.bmp", drawing); // 保存路径图
```

---

### ✅ 6. 返回状态值

```cpp
return state_idle;
```

表示当前状态执行完成，可以切换到空闲状态。

---

## ✅ 二、那这个 `return state_pathPlaning` 是干嘛的？

这不是调用 `pathPlaning()`，而是用于**状态机中的状态返回值**。

---

### 举个例子：

你提到的代码：

```cpp
if (i_count > 6) {
    videoCreate.release();
    this->send(1); // 某种信号
    return state_pathPlaning;
}
```

> 这里并不是在执行 `pathPlaning()`，而是**改变当前程序状态为 `state_pathPlaning`**。

---

### 🎯 状态机会根据这个返回值，在主逻辑中做：

```cpp
switch (state) {
    case state_pathPlaning:
        Auto_Focus_Slect = pathPlaning(); // 实际这里才调用 pathPlaning 函数
        break;
    case state_idle:
        ...
}
```

所以：

* `return state_pathPlaning;` 不是函数调用，而是**告诉主控逻辑“下一步执行 pathPlaning”**；
* 真正调用 `pathPlaning()` 是在状态机的 switch-case 语句里；

---

## ✅ 三、函数调用顺序总图（简化）

```
某些函数中： return state_pathPlaning;
      ↓
主程序状态机检测：if (state == state_pathPlaning)
      ↓
调用 pathPlaning();
      ↓
分析图像 -> 质心坐标 -> 调用 LKH 求路径 -> 保存路径图
```

---

## ✅ 总结：

| 项目                          | 解释                                             |
| --------------------------- | ---------------------------------------------- |
| `pathPlaning()`             | 执行自动路径规划（读取图像 → 找细胞 → 调用 TSP 求路径 → 画路径）        |
| `return state_pathPlaning;` | 表示程序进入路径规划状态，不是函数调用                            |
| 状态机逻辑                       | 在主控函数中根据状态值调用对应函数，如 `pathPlaning()`            |
| 输出文件                        | `Best.txt` 是 LKH 的输出；`cell_8path.bmp` 是路径可视化结果 |

---

如果你想试运行这个路径规划模块，我可以帮你加上控制台打印、弹窗提示、图像显示等功能，要不要我帮你改一版代码，让它的运行过程更清晰？
