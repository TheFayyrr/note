    case state_lensup:
        planeSelection=lensmove(1);//对焦点击驱动向上移动30微米
        break;


//针尖下降
int8_t Pose_Plane:: lensmove(int direction)
{
    emit sendlens(300,direction,0);//驱动物镜上升30microns
    return state_idle;
}
void sendlens(int microstep,int axis,int slowFlag);
connect(decision_task->pose_plane,&Pose_Plane::sendlens,this,&MainWindow::scanningmove);

//扫描区
void MainWindow:: scanningmove(int microstep,int axis,int slowFlag)
{
    if(decision_task->focusFitted)
    {
        if(slowFlag)
        {
            decision_task->patchClampX->priorCmdF((char*)"controller.z.speed.set 15");
            decision_task->patchClampX->focusmove(microstep,axis);
        }
        else
        {
            decision_task->patchClampX->priorCmdF((char*)"controller.z.speed.set 1200");
            decision_task->patchClampX->focusmove(microstep,axis);
        }
    }
}


#  `state_lensup` 状态向量

---

## 🧩 这个状态在干什么？（`state_lensup`）

### ✅ 功能：

在状态机中触发 **物镜（Z轴对焦电机）向上移动 30 微米** 的动作，用于将焦点上移（例如避开某些结构、准备穿刺、或切换图像层）。

---

## 🧭 具体执行流程（状态机逻辑）：

### 💡状态机：

```cpp
case state_lensup:
    planeSelection = lensmove(1);
    break;
```

### 🔧 `lensmove(int direction)` 函数：

```cpp
int8_t Pose_Plane::lensmove(int direction)
{
    emit sendlens(300, direction, 0);  // 发射信号，请求移动 300 步长（即 30 微米），方向 = 1，非慢速
    return state_idle;  // 返回状态为 idle
}
```

* **作用**：发射 `sendlens` 信号，请求移动物镜。
* **参数解释**：

  * `300`: 表示移动步数（通常 1 步 = 0.1 微米 ⇒ 300步 ≈ 30 微米）
  * `direction`: `1` 表示向上（正方向），`-1` 表示向下（负方向）
  * `slowFlag`: 是否慢速移动，`0` 表示快速移动

---

## 🔁 信号与槽连接：

```cpp
connect(decision_task->pose_plane, &Pose_Plane::sendlens, this, &MainWindow::scanningmove);
```

### 🧠 意义：

* **`sendlens(...)` 信号**：在调用 `lensmove()` 函数时发出，通知界面或控制模块：我要移动物镜。
* **`scanningmove(...)` 槽函数**：接收信号后，真正控制对焦电机硬件移动。

---

## ⚙️ `scanningmove(...)` 槽函数分析：

```cpp
void MainWindow::scanningmove(int microstep, int axis, int slowFlag)
{
    if(decision_task->focusFitted)  // 是否完成过自动对焦标志
    {
        if(slowFlag)  // 慢速移动
        {
            decision_task->patchClampX->priorCmdF("controller.z.speed.set 15"); // 设置慢速
            decision_task->patchClampX->focusmove(microstep, axis);             // 发送移动指令
        }
        else           // 快速移动
        {
            decision_task->patchClampX->priorCmdF("controller.z.speed.set 1200"); // 设置快速
            decision_task->patchClampX->focusmove(microstep, axis);               // 发送移动指令
        }
    }
}
```

### ✅ 作用：

1. 根据 `slowFlag` 设置 **对焦电机 Z轴移动速度**。
2. 调用硬件控制接口 `focusmove()` 移动物镜。

### 📝 注意点：

* `patchClampX` 是封装的电机控制器（Prior 控制器）的类对象。
* `priorCmdF(...)` 是向控制器发送指令的函数。
* `focusmove(...)` 是移动 Z 轴的函数。
* `focusFitted == true` 是系统已经对焦过的标志位，确保不会在对焦未完成时移动。

---

## 🧠 整体流程图解（执行链）：

```text
状态机: state_lensup
    ↓
调用: lensmove(1)
    ↓
信号: emit sendlens(300, 1, 0)
    ↓
槽函数: scanningmove(300, 1, 0)
    ↓
控制:
    - 设置速度
    - 控制电机移动 Z 轴
```

---

## ✅ 总结一张表：

| 模块              | 名称               | 功能                             |
| --------------- | ---------------- | ------------------------------ |
| 状态机             | `state_lensup`   | 控制对焦电机上移                       |
| 成员函数            | `lensmove(1)`    | 发出上移信号，默认上移 30 微米              |
| 信号              | `sendlens(...)`  | 请求移动对焦电机                       |
| 槽函数（MainWindow） | `scanningmove()` | 接收到信号后，控制硬件 Z 轴上移（设置速度 + 执行移动） |

---

如果你想要 **再写一个 `state_lensdown`（下移）** 只需调用 `lensmove(-1)` 就行，我也可以帮你补完整个结构。

需要我帮你画个 **流程图** 或者写一个 **自动穿刺上下移动完整模块** 吗？你说一声就行。
