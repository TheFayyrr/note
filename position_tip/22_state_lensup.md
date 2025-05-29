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
# void MainWindow:: scanningmove(int microstep,int axis,int slowFlag)
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


priorCmdF()
# void patchclamp:: focusmove(int addvalue,int axis)//对焦电机相对运动
{
    priorCmdF((char*)"controller.z.position.get");
    std::string str_input(rx);
    std::string extractedDigits = "";
    bool foundMinus = false;
        for (char c : str_input) {
            if (c == '-' && !foundMinus) {
                extractedDigits += c;
                foundMinus = true;
            } else if (isdigit(c)) {
                extractedDigits += c;
            }
        }
        // 将提取的数字字符串转换为整数
    int number = std::stoi(extractedDigits);
    std::cout << number << std::endl;
    int num1= addvalue*axis;//以100个nm为单位
    int sumnum =number+num1;
    std::ostringstream oss;
    string orderstr = "controller.z.goto-position";
    oss << orderstr << ' ' << sumnum;
    string strfinal = oss.str();
    size_t totalLength = strfinal.length() + 1; // +1 for the null terminator
    char* concatenatedStr = new char[totalLength];
     strcpy(concatenatedStr, strfinal.c_str());
    std::cout << concatenatedStr << std::endl;
    priorCmdF(concatenatedStr);
    /*  normally you would poll 'controller.z.busy.get' until response = 0 */
     do
       {
        priorCmdF((char*)"controller.z.busy.get");
        zBusy = atoi(rx);
            } while (zBusy != 0);//判断是否工作完成
    delete[] concatenatedStr;
}
这个就是监控他移动的函数


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

# 在发送信号后 :scanningmove(int microstep, int axis, int slowFlag)
 `state_lensup` 状态下的相关代码链路是如下几层调用：
MainWindow::scanningmove(int microstep, int axis, int slowFlag)
  -> decision_task->patchClampX->priorCmdF(...) // 设定速度
  -> decision_task->patchClampX->focusmove(...) // 移动电机

---

## 🧱 调用链梳理（逻辑顺序）

```cpp
// 状态机调用：
state_lensup:
  -> Pose_Plane::lensmove(direction)
    -> emit sendlens(300, direction, 0)

// Qt 信号槽响应（slot）：
MainWindow::scanningmove(int microstep, int axis, int slowFlag)
  -> decision_task->patchClampX->priorCmdF(...) // 设定速度
  -> decision_task->patchClampX->focusmove(...) // 移动电机

// 发送底层 SDK 命令
patchclamp::priorCmdF(char* tx)
  -> priorSDK2->Cmd(sessionID, tx, rx) // SDK命令执行
```

---

## 🚨 问题关键点：

```cpp
if (!(ret = priorSDK2->Cmd(sessionID,tx,rx)))
    cout << "OK " << rx << endl;
else
    cout << "Api error " << ret << endl;
```

这段代码意味着：

* `priorSDK2->Cmd(...)` 调用失败（返回非 0 错误码）时，就会输出：

  ```cpp
  Api error <ret>
  ```

---

## 🔍 所以问题出在哪里？

这个错误来自底层 SDK `priorSDK2->Cmd(...)` 的调用失败，它可能由 **以下任一原因** 触发：

---

### ✅ 1. `sessionID` 无效（未连接或失效）

* 如果 `sessionID` 是一个错误值（未初始化或超时），SDK 调用自然失败。
* 原因可能包括：

  * `priorSDK2->Open()` 没有调用或失败
  * SDK session 断开或被释放

**建议检查：**

```cpp
cout << "sessionID: " << sessionID << endl;
```

确保它非零且有效。

---

### ✅ 2. 命令格式有误（tx字符串错误）

* 命令字符串 `tx` 如 `"controller.z.speed.set 15"` 是 SDK 命令。
* 如果命令拼写错误、语法错误、参数错误，SDK 也会返回错误。

**建议打印命令内容并核查：**

```cpp
cout << "Sending command: " << tx << endl;
```

---

### ✅ 3. SDK 初始化失败或 `priorSDK2` 为 null

* 如果 `priorSDK2` 指针本身没有正确初始化，那么调用 `Cmd()` 就不可靠。
* 特别是在多线程 Qt 程序中，容易出现在对象生命周期管理不当的情况。

**建议添加断言或空指针检查：**

```cpp
if (!priorSDK2) {
    cerr << "Error: priorSDK2 is null!" << endl;
    return -1;
}
```

---

### ✅ 4. 硬件未连接或串口失效

* 如果 Prior 控制器未通电或未连接，底层 `Cmd()` 无法完成。
* 通信端口错误也会造成调用失败。

**建议：**

* 检查 Prior 控制器是否通过 USB/串口正确连接
* 控制器是否开启
* 程序是否以管理员权限运行（某些串口访问需要）

---

## 🧪 调试建议步骤：

1. **打印 sessionID**，确认非零且有效。
2. **打印 tx 命令字符串**，确认命令拼写、参数无误。
3. **检查 priorSDK2 是否为 NULL**。
4. **确认控制器是否正常连接、供电**。
5. **SDK 返回码 ret 打印更详细信息**（可查 Prior SDK 文档）。

---

## ✅ 示例调试增强建议：

```cpp
int patchclamp::priorCmdF(char *tx) 
{
    if (!priorSDK2) {
        cerr << "priorSDK2 is null!" << endl;
        return -1;
    }

    if (sessionID == 0) {
        cerr << "Invalid sessionID!" << endl;
        return -2;
    }

    cout << "Sending command: " << tx << " to session: " << sessionID << endl;
    
    ret = priorSDK2->Cmd(sessionID, tx, rx);
    if (!ret)
        cout << "OK: " << rx << endl;
    else
        cout << "Api error " << ret << " from command: " << tx << endl;

    return ret;
}
```

---

## 🔚 总结一句话：

> 如果你看到输出 `Api error <ret>`，那么问题几乎一定出在 **Prior 控制器 SDK 的底层调用失败**，主要可能是 **sessionID 无效、命令字符串错误、priorSDK2 未初始化、控制器未连接** 四类问题。

需要你重点检查初始化流程是否执行成功，连接是否有效。你可以把相关初始化代码贴出来，我可以帮你一并梳理并定位问题源头。


# 查询状态
你这段输出内容其实是在反映 **物镜 Z 轴（controller.z）移动指令的执行与状态查询过程**。我来一步步给你解释它的含义和运行位置。

---

## 📌 1. 初始化阶段：

```plaintext
focuspositionget is 0 inital plane
```

* 表示你初始化读取了 Z 轴位置，得到初始位置是 `0`。
* 这可能是通过命令：`controller.z.position.get` 得到的。

---

## 📌 2. 设置速度与移动指令发送：

```plaintext
controller.z.speed.set 1200en
s move up 30 microns
OK 0
controller.z.position.get
OK 0
controller.z.goto-position 300
controller.z.goto-position 300
OK 0
```

### 解释：

* `controller.z.speed.set 1200`：设定 Z 轴的移动速度为 1200。
* `controller.z.goto-position 300`：控制器尝试将物镜移动到 300（单位是微米）。
* `OK 0`：表示指令发送成功，返回状态码 0（无错）。

---

## 📌 3. 移动状态查询阶段：

```plaintext
controller.z.busy.get
OK 4
controller.z.busy.get
OK 4
...
controller.z.busy.get
OK 0
```

### 解释：

这是 **查询 Z 轴是否正在移动** 的命令：`controller.z.busy.get`

* `OK 4` 表示 **Z 轴“忙碌中”**（正在运动）。
* 连续出现多个 `OK 4` 是轮询等待完成移动的过程（这段代码会在一个循环中反复查询）。
* 最终出现 `OK 0`，表示 **Z 轴已经停止移动（空闲）**，运动完成。

---

## 🧠 运行位置是在哪？

你这段打印来自于控制器运动逻辑的部分，可能的代码类似于如下：

```cpp
// 设置速度
priorCmdF("controller.z.speed.set 1200");

// 移动到某个位置
priorCmdF("controller.z.goto-position 300");

// 查询忙碌状态
do {
    priorCmdF("controller.z.busy.get");
    Sleep(100); // 等待 100ms（伪代码）
} while (返回的是 4);
```

这通常在 `patchclamp::focusmove(...)` 或类似函数中调用，用于实现一个**阻塞式等待移动完成**的逻辑。

---

## 🧪 小结一下：

| 行为     | 命令                               | 含义                 |
| ------ | -------------------------------- | ------------------ |
| 获取当前位置 | `controller.z.position.get`      | 得到当前 Z 轴位置（如 0）    |
| 设置速度   | `controller.z.speed.set 1200`    | 设置运动速度             |
| 执行运动   | `controller.z.goto-position 300` | 发送目标位置             |
| 监控状态   | `controller.z.busy.get`          | 返回 4 表示正在移动，0 表示完成 |

---

## ✅ 结论：

你贴的这段输出表示：

> 控制器成功接受了移动到 300 微米的命令，并在接下来的几十个循环中处于“移动中（busy=4）”状态，直到最终返回“busy=0”，表示移动完成。

---

如果你还有 `focusmove` 或 `goto-position` 的代码片段可以贴出来，我可以更精确地定位这段行为是在哪个函数内部运行的。也可以帮你检查是否有必要优化这个“忙等”轮询结构（例如用超时机制、防止卡死）。
#  rx与tx 指的是什么？

你问得非常关键，咱们来逐层解析你的代码运行机制，尤其是 `controller.z.busy.get` 一直在输出的来源问题。

---

## 🔧 `Cmd(...)` 函数分析

你提供的这个函数：

```cpp
int Cmd(int sessionID, const char *const tx, char *const rx)
{
    return PriorScientificSDK_cmd(sessionID, tx, rx);
}
```

是你系统中 **与 Prior Scientific 硬件通信的最底层函数调用**。我们来分解一下：

| 参数          | 类型            | 含义                         |
| ----------- | ------------- | -------------------------- |
| `sessionID` | `int`         | 表示与控制器通信的会话 ID（连接句柄）       |
| `tx`        | `const char*` | **你要发给硬件的命令字符串**（transmit） |
| `rx`        | `char*`       | **硬件返回的响应结果**（receive）     |

所以 `tx` 是发送命令，`rx` 是接收结果。

例如：

```cpp
char rx[256];
Cmd(sessionID, "controller.z.busy.get", rx);
```

执行完后，`rx` 中就可能包含返回值如：`"OK 4"` 或 `"OK 0"`。

---

## 🔄 你看到的“持续输出 busy=4”的根本原因是：

你的某段程序在一个循环中不停调用：

```cpp
priorCmdF("controller.z.busy.get");
```

而 `priorCmdF` 本身大概率是这样的：

```cpp
int patchclamp::priorCmdF(char *tx)
{
    cout << tx << endl; // 打印你发出的命令
    if (!(ret = priorSDK2->Cmd(sessionID, tx, rx)))
        cout << "OK " << rx << endl;
    else
        cout << "Api error " << ret << endl;
    return 0;
}
```

也就是说：

* 每次你看到 `controller.z.busy.get` 输出是因为你打了 `cout << tx`
* 每次你看到 `OK 4` 是因为你打了 `cout << "OK " << rx`

🔁 这些被反复打印，说明有代码在**不断循环调用 `priorCmdF("controller.z.busy.get")`** 来**轮询是否移动完成**。

---

## 🔍 那这段循环在哪里？

它通常存在于你控制 Z 轴移动的函数，比如你可能有类似这样的逻辑：

```cpp
void patchclamp::focusmove(int microstep, int axis)
{
    char tx[256], rx[256];
    
    // 例如设定位置
    sprintf(tx, "controller.z.goto-position %d", microstep);
    Cmd(sessionID, tx, rx);
    
    // 等待移动完成
    while (true)
    {
        Cmd(sessionID, "controller.z.busy.get", rx);
        if (strcmp(rx, "0") == 0 || strcmp(rx, "OK 0") == 0)
            break;
        Sleep(100); // 停一下
    }
}
```

> 💡 如果你找不到这段代码，搜索关键词 `"controller.z.busy.get"` 或 `"priorCmdF"` 看是谁在循环里一直调用它。

---

## ✅ 总结你现在看到的机制：

1. 你调用了 `lensmove` 或 `scanningmove` → 内部调用了 `focusmove` → 发出了移动命令。
2. 在 `focusmove` 内部或其它地方有个循环调用 `priorCmdF("controller.z.busy.get")`。
3. 每次调用都把 `tx` 和 `rx` 打印出来。
4. 所以你在控制台看到持续输出的 `controller.z.busy.get` 和 `OK 4` 是这个轮询循环造成的。

---

## ✅ 关键定义小结：

| 名称                      | 意义                                       |
| ----------------------- | ---------------------------------------- |
| `tx`                    | transmit，发送给控制器的字符串命令                    |
| `rx`                    | receive，从控制器读取到的响应内容                     |
| `Cmd(...)`              | 使用 `PriorScientificSDK_cmd` 和控制器进行一次命令传输 |
| `priorCmdF(...)`        | 封装了 `Cmd` 并输出调试日志的辅助函数                   |
| `controller.z.busy.get` | 查询 Z 轴是否正在移动，返回 `OK 4` 表示忙，`OK 0` 表示空闲   |

---

如果你愿意贴一下 `focusmove` 或 `gotoZ` 等控制移动的函数代码，我可以准确指出这段轮询在哪儿。你很可能想给它加个超时保护逻辑，防止控制器卡死后无限循环。需要我帮你加一下也可以。

# focusmove的代码
非常清晰，感谢你贴出 `focusmove()` 的完整代码。我们现在可以 **100% 确定**，你观察到的 `controller.z.busy.get` 的持续输出正是来自这段代码中的轮询逻辑：

---

## 🔍 关键点：轮询控制器忙状态的位置

在你的 `focusmove()` 函数中，有如下代码：

```cpp
do {
    priorCmdF((char*)"controller.z.busy.get");
    zBusy = atoi(rx);
} while (zBusy != 0);
```

* 这一段是 **在控制器移动镜头后，轮询 `busy` 状态是否为 0**。
* 每一次循环会调用 `priorCmdF("controller.z.busy.get")`，它会：

  * 发送命令字符串 `tx = "controller.z.busy.get"`
  * 接收响应到 `rx`
  * 打印 `tx` 和 `rx`，你就看到了一连串的输出。

---

## 📌 为什么你会看到大量 `OK 4`？

原因很简单：**控制器还在移动中，busy 状态是 4**，不等于 0，循环就继续跑下去。

直到最后某次，控制器完成移动，返回：

```txt
OK 0
```

循环才会退出，执行 `delete[] concatenatedStr;` 并结束函数。

---

## ⚠️ 建议添加超时保护逻辑（防止死循环）

目前你这个 `do...while` 是 **死循环**，万一控制器异常（比如线松、通信丢失、控制器没有清零），你就会**卡死在这里**，永远打印 `OK 4`。

你可以修改为如下形式加个超时次数限制，例如最多等 5 秒，每 100ms 轮询一次：

```cpp
int maxAttempts = 50; // 最多尝试50次，5秒
int attempts = 0;
do {
    priorCmdF((char*)"controller.z.busy.get");
    zBusy = atoi(rx);
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    attempts++;
    if (attempts >= maxAttempts) {
        std::cerr << "[Error] Timeout waiting for controller.z.busy.get == 0\n";
        break; // 或者 return error_code;
    }
} while (zBusy != 0);
```

你需要 `#include <thread>` 和 `#include <chrono>` 来支持这个延迟语句。

---

## ✅ 总结你代码行为

| 行为                               | 解释                                |
| -------------------------------- | --------------------------------- |
| `controller.z.position.get`      | 获取当前位置，用于计算目标位置                   |
| `controller.z.goto-position xxx` | 控制镜头移动到新位置                        |
| `controller.z.busy.get`          | 持续轮询控制器是否完成移动，打印出 `OK 4` 或 `OK 0` |
| 持续打印 `OK 4`                      | 表示镜头正在移动中                         |
| 最后打印 `OK 0`                      | 表示镜头移动完成，退出循环                     |

---

如果你愿意，我可以帮你进一步整理这段 `focusmove()` 的代码，包括内存安全（避免 `new/delete`）、错误处理和可读性提升。需要我优化一下吗？

# priorCmd
你的疑问很关键，我们来一步步梳理这两个函数的区别与作用：

---

## 🧠 先说说两个函数分别是啥

### ✅ 1. `priorCmdF(char *tx)`

这是你自己写的一个“命令发送函数”，封装了调用底层 SDK 的命令接口。

```cpp
int patchclamp::priorCmdF(char *tx) {
    cout << tx << endl;  // 打印发送的命令
    if (!(ret = priorSDK2->Cmd(sessionID, tx, rx)))
        cout << "OK " << rx << endl; // 接收返回结果 rx
    else
        cout << "Api error " << ret << endl;
    return 0;
}
```

📌 作用是：

* 把字符串命令（tx）发给 Prior 控制器
* 接收返回值（rx 存结果，比如位置值、busy 状态等）
* 打印发送与接收内容，方便调试

这个是你和设备通信的“核心函数”。

---

### ✅ 2. `focusmove(int addvalue, int axis)`

这是你写的高层封装函数，用来控制“Z 轴焦距”的相对运动。

它的行为更复杂，是对多个 `priorCmdF()` 的组合调用：

```cpp
void patchclamp::focusmove(int addvalue, int axis)
{
    // Step 1: 先获取当前 Z 轴位置
    priorCmdF("controller.z.position.get");

    // Step 2: 从 rx 中解析出位置数字
    // 提取字符串中的数字（即当前位置）
    std::string str_input(rx);
    int number = 解析出的数字;

    // Step 3: 计算新目标位置 = 当前 + 增量
    int sumnum = number + addvalue * axis;

    // Step 4: 构造 goto 命令字符串
    string cmd = "controller.z.goto-position 目标位置";
    priorCmdF(cmd);

    // Step 5: 轮询 busy 状态是否完成
    do {
        priorCmdF("controller.z.busy.get");
    } while (zBusy != 0);
}
```

📌 作用是：

* 自动读取当前位置
* 加上偏移量后，生成目标位置命令
* 下发 `goto-position` 命令
* 一直轮询 `busy.get` 判断是否完成

---

## 🧩 两者关系：**底层命令接口 vs 高层控制逻辑**

| 函数名           | 角色     | 功能说明                                  |
| ------------- | ------ | ------------------------------------- |
| `priorCmdF()` | 底层通信接口 | 单次发送命令给 Prior 控制器，并把响应放入 `rx` 中       |
| `focusmove()` | 高层运动封装 | 多次调用 `priorCmdF()` 来实现“焦距相对运动 + 完成等待” |

可以这么理解：

> `focusmove()` 是你写的业务逻辑函数，依赖于 `priorCmdF()` 来完成具体动作。它是把一系列硬件命令组织起来实现某个实际功能。

---

## 💡 一句话总结

> `priorCmdF()` 是“发命令、收回应”的小兵；
> `focusmove()` 是“控制流程”的将军，调兵遣将，用小兵干活。

---

如果你还是不太清楚 `std::string` 和 `char*` 的拼接/转换，也可以问我，我可以帮你逐行讲解 C++ 字符串拼接部分的含义。


# 细拆一遍 `focusmove()` 的每一行做了什么
好的，我们来**逐行剖析你写的 `focusmove()` 函数**，让你完全搞懂每一步的作用：

---

```cpp
void patchclamp::focusmove(int addvalue, int axis)
```

🔹 这是类 `patchclamp` 的成员函数，名字叫 `focusmove`（焦距移动），接受两个参数：

* `addvalue`: 每次移动的步长（比如 300）
* `axis`: 方向（通常是 +1 向上，-1 向下）

---

```cpp
priorCmdF((char*)"controller.z.position.get");
```

🔹 向 Prior 控制器发送命令 `"controller.z.position.get"`
意思是：**获取当前 Z 轴的位置**
结果会存入全局 `rx` 缓冲区中，比如 `"OK 0"`、`"OK 123"` 之类。

---

```cpp
std::string str_input(rx);
std::string extractedDigits = "";
bool foundMinus = false;
```

🔹 把 `rx`（返回的 char 数组）转换为 `std::string` 类型，方便后面处理
🔹 初始化两个变量用于提取数字字符串：

* `extractedDigits` 用于拼接数字字符
* `foundMinus` 用于判断有没有负号（只保留第一个）

---

```cpp
for (char c : str_input) {
    if (c == '-' && !foundMinus) {
        extractedDigits += c;
        foundMinus = true;
    } else if (isdigit(c)) {
        extractedDigits += c;
    }
}
```

🔹 遍历返回字符串，**只保留第一个负号和所有数字**
例如：

* 输入是 `"OK -130"` ➜ 提取出 `"-130"`
* 输入是 `"OK 245"` ➜ 提取出 `"245"`

---

```cpp
int number = std::stoi(extractedDigits);
```

🔹 把提取出来的数字字符串转换成整数，表示 **当前位置**

---

```cpp
std::cout << number << std::endl;
```

🔹 打印当前位置，便于调试

---

```cpp
int num1 = addvalue * axis;
int sumnum = number + num1;
```

🔹 `num1` 是位移量（可以正/负）
🔹 `sumnum` 是 **新的目标位置**

例子：

* 当前为 300，`addvalue=100`，`axis=1` ➜ 新位置是 400
* 当前为 300，`addvalue=100`，`axis=-1` ➜ 新位置是 200

---

```cpp
std::ostringstream oss;
string orderstr = "controller.z.goto-position";
oss << orderstr << ' ' << sumnum;
```

🔹 构造目标命令字符串，比如：

```
"controller.z.goto-position 400"
```

这是要下发的目标位置命令。

---

```cpp
string strfinal = oss.str();
size_t totalLength = strfinal.length() + 1; // +1 for the null terminator
char* concatenatedStr = new char[totalLength];
strcpy(concatenatedStr, strfinal.c_str());
```

🔹 把 C++ 字符串 `strfinal` 转换成 C 风格字符串 `char*`，以便用于 `priorCmdF()` 的接口（因为它接收 `char*`）

---

```cpp
std::cout << concatenatedStr << std::endl;
```

🔹 打印最终构造的命令字符串，例如：

```
controller.z.goto-position 400
```

---

```cpp
priorCmdF(concatenatedStr);
```

🔹 把命令发送出去，驱动 Z 轴运动到目标位置

---

```cpp
do {
    priorCmdF((char*)"controller.z.busy.get");
    zBusy = atoi(rx);
} while (zBusy != 0);
```

🔹 用 busy 状态轮询判断运动是否完成：

* 每次发出 `"controller.z.busy.get"` 命令
* 返回值 `rx` 中会有 `0` 或 `4`

  * `0`: 空闲（运动完成）
  * `4`: 忙碌（正在运动）

🔁 如果返回的是 4，就继续等待

---

```cpp
delete[] concatenatedStr;
```

🔹 手动释放刚才 `new` 出来的动态字符串内存，防止内存泄漏

---

## ✅ 总结一句话

这段代码实现了 **Z 轴移动到目标位置，并等待它完成**，你可以理解为：

```
目标位置 = 当前 + 相对位移
发送 goto-position 命令
轮询 busy 状态直到完成
```

---

如果你还想进一步优化，比如不用 `new/delete` 手动分配内存，我也可以教你怎么用 `std::vector<char>` 或 `std::unique_ptr<char[]>` 替代，写出更现代安全的 C++ 代码。需要的话随时说。
