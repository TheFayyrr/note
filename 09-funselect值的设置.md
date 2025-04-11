我来帮你搜索整个项目中对 funSelect 的赋值和改变。让我根据代码分析来解释 funSelect 的值是如何被设置和改变的：

1. 初始值设置：
```cpp
int8_t funSelect=state_init;  // 在 Decision_Task 类中定义初始值
```

2. 主要通过以下几种方式改变 funSelect 的值：

A. UI界面按钮触发：
- `on_ImageAdjust_clicked()`: `decision_task->funSelect=7` //图片自动校准
- `on_NextAction_clicked()`: `decision_task->funSelect=4` //半自动模式
- `on_fullyNextAct_clicked()`: `decision_task->funSelect=11` //接触独立测试
- `on_testOpen_clicked()`: `decision_task->funSelect=12`
- `on_distanceSend_clicked()`: `decision_task->funSelect=9`

B. Auto_Focus 完成时的信号槽连接：
```cpp
connect(A_Focus, &Auto_Focus::afOver, this, &Decision_Task::decisionSelect);
```
当 Auto_Focus 发出 afOver 信号时，会调用 decisionSelect 函数更改 funSelect 的值。

C. 在 decision_task_run() 函数内部也会修改 funSelect：
```cpp
case 100:
    timer=new QTimer();
    timer->setInterval(50);
    connect(timer,&QTimer::timeout,this,&Decision_Task::decision_task_run);
    timer->start();
    funSelect=state_idle;  // 设置为空闲状态
    break;

case 4:  // 半自动模式
    // ... 执行操作 ...
    funSelect=0;  // 任务完成后重置
    break;
```

3. funSelect 的值会影响系统的工作流程：
- 在 decision_task_run() 中通过 switch-case 结构根据 funSelect 的值执行不同的操作
- 每个 case 分支代表一种工作模式，比如：
  - case 4: 半自动模式
  - case 7: 图像自动校准
  - case 11: 接触独立测试
  - case 12: gigaseal 模式

4. 定时器驱动：
```cpp
timer=new QTimer();
timer->setInterval(50);  // 50ms 间隔
connect(timer,&QTimer::timeout,this,&Decision_Task::decision_task_run);
```
系统通过定时器每50ms调用一次 decision_task_run()，检查并执行对应的任务。

所以 funSelect 实际上是一个状态机控制变量，通过 UI 操作、自动聚焦完成信号、以及内部任务完成等多个途径来改变其值，从而控制整个系统的工作流程和状态转换。这样的设计允许系统在不同的操作模式之间切换，并保持任务的连续性。
-----------------------------------------------------------------------------------------------

让我分析一下所有的 funSelect 使用情况：让我总结一下对 funSelect 和 case 4 的分析：

case 4 是系统最主要的半自动模式，但它并不是唯一的功能入口。通过分析，系统的主要功能通过不同的 funSelect 值来控制：

1. case 4: 半自动模式（主要工作模式）
   - 实现针尖定位、接触检测、穿刺等核心功能
   - 通过 stateHandle 数组控制工作流程
   - 可以通过 NextAction 按钮触发(funSelect=4)

2. 其他重要功能入口：
   - case 1: 全自动模式 (通过 fullyStart 按钮)
   - case 5-8: 图像采集和处理相关功能
   - case 9: 半自动活检模式
   - case 11: 独立接触检测模式
   - case 12: gigaseal 模式(膜片钳)

funSelect 的值可以通过以下方式改变：
1. UI界面按钮直接设置:
```cpp
void MainWindow::on_ImageAdjust_clicked() {
    decision_task->funSelect=7;  // 图片自动校准
}
```

2. 通过 decisionSelect 函数设置:
```cpp
void Decision_Task::decisionSelect(int8_t select) {
    funSelect=select;
}
```

3. 任务完成后自动重置:
```cpp
case 4:
    // ...执行操作...
    funSelect=0;  // 任务完成后重置
```

所以虽然 case 4 是最主要的工作模式，但完整的系统功能需要通过其他 funSelect 值来实现：
- 图像处理(case 5-8)
- 接触检测(case 11)
- 膜片钳操作(case 12)
- 全自动模式(case 1)
等都是独立且必要的功能。

每个 case 都有其特定的用途和工作流程，它们共同构成了完整的系统功能。根据具体的实验需求，需要配合使用不同的功能模式。
_______________________________________________________________________
让我分析一下所有的 funSelect 使用情况：让我总结一下对 funSelect 和 case 4 的分析：

case 4 是系统最主要的半自动模式，但它并不是唯一的功能入口。通过分析，系统的主要功能通过不同的 funSelect 值来控制：

1. case 4: 半自动模式（主要工作模式）
   - 实现针尖定位、接触检测、穿刺等核心功能
   - 通过 stateHandle 数组控制工作流程
   - 可以通过 NextAction 按钮触发(funSelect=4)

2. 其他重要功能入口：
   - case 1: 全自动模式 (通过 fullyStart 按钮)
   - case 5-8: 图像采集和处理相关功能
   - case 9: 半自动活检模式
   - case 11: 独立接触检测模式
   - case 12: gigaseal 模式(膜片钳)

funSelect 的值可以通过以下方式改变：
1. UI界面按钮直接设置:
```cpp
void MainWindow::on_ImageAdjust_clicked() {
    decision_task->funSelect=7;  // 图片自动校准
}
```

2. 通过 decisionSelect 函数设置:
```cpp
void Decision_Task::decisionSelect(int8_t select) {
    funSelect=select;
}
```

3. 任务完成后自动重置:
```cpp
case 4:
    // ...执行操作...
    funSelect=0;  // 任务完成后重置
```

所以虽然 case 4 是最主要的工作模式，但完整的系统功能需要通过其他 funSelect 值来实现：
- 图像处理(case 5-8)
- 接触检测(case 11)
- 膜片钳操作(case 12)
- 全自动模式(case 1)
等都是独立且必要的功能。

每个 case 都有其特定的用途和工作流程，它们共同构成了完整的系统功能。根据具体的实验需求，需要配合使用不同的功能模式。
