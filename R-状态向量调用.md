这是我的问题： vector<Point2i> stateHandleSemi={
            {state_idle,state_biopsypointrecord},
            {state_idle,state_tipseg},
            {state_idle,state_movebiopsypoint},
            {state_idle,state_pointclick},
            {state_idle,state_touchDetection},
            {state_idle,state_pointclick},
            {state_idle,state_touchDetection},
        {state_idle,state_pointclick},
        {state_idle,state_touchDetection},
        {state_idle,state_pointclick},
        {state_idle,state_touchDetection},
        {state_idle,state_pointclick},
        {state_idle,state_touchDetection},
        {state_idle,state_pointclick},
        {state_idle,state_touchDetection},
        {state_idle,state_pointclick},
        {state_idle,state_touchDetection},
        {state_idle,state_pointclick},
        {state_idle,state_touchDetection}};//其中x是细胞的状态，y是针尖的状态,
        状态向量是如何被调用的呢？比如上面这个状态向量stateHandleSemi，他是在哪里赋值给函数的呢？怎么被调用执行的呢？而且，decision_task中有好几个状态向量，你怎么知道，是调用的哪个状态向量的呢？，其次，我理解的是通过改变funselect来决策decision_task中的decision函数，根据case值，在调用其他的文件里的函数，position_cell中的Pose_Plane::decision()，由planeSelection决定case值调用函数，position_tip 的Auto_Focus::decision() 由Auto_Focus_Slect根据case值，在调用其他的文件里的函数。但是，我产生另一个疑问：decision_taskcase是具体的数字，而position_tip和position_cell文件中的 case值是state_函数名，并不是具体数字，这个是为什么呢？而且这都没用到状态向量stateHandleSemi，而且调用函数的返回值也没有保存，这不是相当于没有调用了吗？
        你上一条回答好像不对，
——————————————————————————————————————————————————————————————————————————————————————————————————————————————————————
让我重新解释状态向量的调用机制，特别是针对 `stateHandleSemi`：

# 1. 状态向量的调用流程

```mermaid
flowchart TD
    A[MainWindow] -->|decisionSelect(4)| B[Decision_Task]
    B -->|case 4| C[decision_task_run]
    C -->|actionStep| D[选择状态向量]
    D -->|stateHandleSemi| E[设置模块状态]
```

# 2. 具体调用代码分析

在 decision_task.cpp 中的关键部分：
```cpp
void Decision_Task::decision_task_run() {
    switch (funSelect) {
        case 4:  // 半自动模式
            A_Focus->AutoOpen = 0;
            pose_plane->AutoOpen = 0;
            
            // 重要：这里选择使用哪个状态向量
            if(actionStep%21-1 >= 0) {
                // 设置各模块的状态
                A_Focus->Auto_Focus_Slect = stateHandleSemi[actionStep%21-1].x;
                pose_plane->planeSelection = stateHandleSemi[actionStep%21-1].y;
            }
            actionStep++;
            break;
    }
}
```

# 3. 状态向量选择机制

- `funSelect = 4` 时使用 `stateHandleSemi`
- `funSelect = 1` 时使用 `stateHandle`
- `funSelect = 8` 时使用 `stateHandleMultibiopsy`

# 4. 状态值映射说明

```cpp
// decision_task.cpp 中的 case 值
case 1:  // 全自动模式
case 4:  // 半自动模式
case 8:  // 多点活检模式

// positioning_tip.h 中的枚举值
enum tip_States {
    state_touchDetection = 3,    // 编译后变为 3
    state_pointclick = 20,       // 编译后变为 20
    // ...
}
```

状态值在编译时会被转换为对应的数字，所以实际执行时都是数字。

# 5. 执行流程示例

```cpp
// 1. MainWindow触发
void MainWindow::on_startButton_clicked() {
    decision_task->decisionSelect(4);  // 选择半自动模式
}

// 2. Decision_Task执行
void Decision_Task::decision_task_run() {
    case 4:
        // 使用stateHandleSemi设置状态
        pose_plane->planeSelection = stateHandleSemi[actionStep].y;
        // 例如: state_biopsypointrecord (值为25)
}

// 3. Pose_Plane执行
void Pose_Plane::decision() {
    switch(planeSelection) {
        case state_biopsypointrecord:  // case 25:
            ret = recordbiopsypoint();  // 执行具体功能
            if(ret == 1) {
                planeSelection = state_idle;  // 执行完成，状态复位
            }
            break;
    }
}
```

# 6. 返回值处理

实际上，返回值是通过状态变化隐式处理的：
```cpp
void Decision_Task::decision_task_run() {
    if(A_Focus->Auto_Focus_Slect == state_idle && 
       pose_plane->planeSelection == state_idle) {
        // 当前步骤完成，进入下一步
        actionStep++;
    }
    // 否则继续执行当前步骤
}
```

这种设计使用状态机模式：
1. 状态向量定义任务序列
2. `funSelect` 选择工作模式
3. `actionStep` 控制执行进度
4. 通过状态变化（而不是返回值）控制流程
