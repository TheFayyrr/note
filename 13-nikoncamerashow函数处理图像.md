# nikoncamerashow()` 函数是在 Pose_Kalman 类的函数

让我帮您分析这些连接关系：

1. `nikoncamerashow()` 函数是在 Pose_Kalman 类的 decision() 函数中通过 case 8 调用的：
```cpp
case 8:
   nikoncamerashow();
   break;
```

2. 让我搜索谁接收了 sendImage 信号：基于搜索结果，我来完整解释这个函数的所有关联：

1. 发送端 - nikoncamerashow() 函数最后通过 emit sendImage(outImg) 发送图像

2. 接收端 - 在 MainWindow 构造函数中设置了多个接收连接:
```cpp
// MainWindow 构造函数中:
connect(decision_task->pose_plane, &Pose_Plane::sendImage, this, &MainWindow::showImage);
connect(decision_task->imageCollect, &Pose_Kalman::sendImage, this, &MainWindow::showImage);
connect(decision_task->A_Focus, &Auto_Focus::sendImage, this, &MainWindow::showImage);
```

3. 相关按钮和功能：
- capture 按钮 (`on_capture_clicked`): 设置 funSelect=8 来启动显示
- format 改变 (`formatchange`): 设置 funSelect=8 来刷新显示
- ROI区域显示 (`roishow`): 设置 funSelect=8 来更新显示
- 录制开始时: 设置 funSelect=8
- 图像保存相关功能：通过 saveFlag、clusterFlag、mitoFlagB、mitoFlagF 等标志控制

4. 图像处理功能：
- CLAHE 图像增强 (`on_CLAHE_clicked`)
- 对比度调整 (`sendcontrast`)
- 自动曝光 (`on_Autoexposure_clicked`)
- 白平衡 (`on_Blancewhite_clicked`)

5. 显示流程：
- nikoncamerashow() 捕获图像
- 处理图像(根据各种标志进行不同处理)
- 转换为 QImage
- 发送信号
- MainWindow::showImage 接收并通过 videopainter 显示

这是一个典型的 Qt 信号槽机制的应用，通过 funSelect=8 触发 nikoncamerashow()，然后通过信号槽机制将图像传递给主窗口显示。多个对象(pose_plane、imageCollect、A_Focus)都可以发送图像，统一由 MainWindow::showImage 处理显示。
