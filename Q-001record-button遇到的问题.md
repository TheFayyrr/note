''''
void Pose_Kalman::recordVideo()
{
    Mat tansImg,trainimg;
    
    if(cameraOpen) {
        // 获取相机图像
        while(cameracontrol->grabImage(trainimg)==0) {
            /*break;*/
        }
        
        // 根据录制状态处理
        if(recordFlag == 1) {  // 开始录制
            if(videoWriter == nullptr) {
                QString savePath = videoSavePath.isEmpty() ? QString("./") : videoSavePath;
                QString fileName = videoFileName.isEmpty() ? QString("recording.avi") : videoFileName;
                
                // 创建保存目录
                QDir dir(savePath);
                if(!dir.exists()) {
                    dir.mkpath(".");
                }
                
                QString fullPath = savePath + "/" + fileName;
                std::cout << "开始录制视频,保存至: " << fullPath.toStdString() << std::endl;
                
                // 使用 MJPG 编码器
                videoWriter = new VideoWriter(fullPath.toStdString(), 
                    VideoWriter::fourcc('M','J','P','G'), 25, Size(4096,2168), true);
                
                if(!videoWriter->isOpened()) {
                    std::cout << "创建视频文件失败!" << std::endl;
                    delete videoWriter;
                    videoWriter = nullptr;
                    recordFlag = -1;
                    return;
                }
                frameCount = 0;
            }
            
            // 写入当前帧
            tansImg = trainimg.clone();
            videoWriter->write(tansImg);
            frameCount++;
            std::cout << "已录制 " << frameCount << " 帧" << std::endl;
            
        } else if(recordFlag == 0) {  // 停止录制
            if(videoWriter != nullptr) {
                std::cout << "正在保存视频,共 " << frameCount << " 帧..." << std::endl;
                videoWriter->release();
                delete videoWriter;
                videoWriter = nullptr;
                std::cout << "视频保存完成!" << std::endl;
                frameCount = 0;
            }
            recordFlag = -1;
        }
    }
}
''''
我采用的是这个录制视频的函数，但是他录制的视频有问题打不开，我怀疑是录的那个灰色的视频，就灰的图像视频和那个彩色图像视频，对应关系不对，就是比如说你你往里面输的是彩色的，但是它那个视频你录制是灰色的，这个对应关系，你看一下是不是弄错?

-----------------------------------------------
明白了,问题可能出在录制视频时没有正确处理图像的颜色格式。让我们检查一下 cameracontrol->grabImage 返回的图像格式,并确保正确处理颜色空间。我查看了代码,发现相机返回的图像可能是灰度格式,但我们在视频写入时使用了彩色模式。让我修改recordVideo函数来正确处理图像格式:
问题出现在大小格式上。
````
void Pose_Kalman::recordVideo()
{
    Mat imageRead, tansImg;
    static int numFlag = 0;
    static bool isVideoWriterInitialized = false;
    static VideoWriter videoCreate;

    if(cameraOpen) {
        // 获取相机图像
        while(cameracontrol->grabImage(imageRead) == 0) {
            /*break;*/
        }

        if(!imageRead.empty()) {
            if(recordFlag == 1) {
                if(!isVideoWriterInitialized) {
                    // 获取实际图像尺寸和通道数
                    Size imgSize = imageRead.size();
                    int channels = imageRead.channels();
                    std::cout << "图像尺寸: " << imgSize.width << "x" << imgSize.height 
                              << " 通道数: " << channels << std::endl;
                    
                    // 创建视频写入器
                    QString savePath = videoSavePath.isEmpty() ? QString("./") : videoSavePath;
                    QString fileName = videoFileName.isEmpty() ? QString("recording.avi") : videoFileName;
                    QString fullPath = savePath + "/" + fileName;
                    
                    // 如果是灰度图像,需要转换为BGR
                    bool isColor = (channels == 3);
                    videoCreate.open(fullPath.toStdString(), 
                        VideoWriter::fourcc('M','J','P','G'), 25, imgSize, isColor);
                    
                    if(!videoCreate.isOpened()) {
                        std::cout << "无法创建视频文件!" << std::endl;
                        recordFlag = -1;
                        return;
                    }
                    isVideoWriterInitialized = true;
                    std::cout << "开始录制视频,保存至: " << fullPath.toStdString() << std::endl;
                }
                
                // 写入当前帧
                tansImg = imageRead.clone();
                if(imageRead.channels() == 1) {
                    // 如果是灰度图像,转换为BGR
                    cvtColor(tansImg, tansImg, COLOR_GRAY2BGR);
                }
                videoCreate.write(tansImg);
                numFlag++;
                std::cout << "已录制 " << numFlag << " 帧" << std::endl;
            } 
            else if(recordFlag == 0) {
                if(isVideoWriterInitialized) {
                    std::cout << "正在保存视频,共 " << numFlag << " 帧..." << std::endl;
                    videoCreate.release();
                    isVideoWriterInitialized = false;
                    std::cout << "视频保存完成!" << std::endl;
                    numFlag = 0;
                }
                recordFlag = -1;
            }
        } else {
            std::cout << "获取图像失败!" << std::endl;
        }
    }
}
````
---------------------------------------------


已进行更改。

我对 recordVideo 函数进行了以下关键修改:

1. 添加了对图像通道数的检测 (imageRead.channels())
2. 根据实际图像格式来正确初始化 VideoWriter:
   - 如果是灰度图像(1通道),设置 isColor = false
   - 如果是彩色图像(3通道),设置 isColor = true
3. 对于灰度图像,在写入前先转换为 BGR 格式: cvtColor(tansImg, tansImg, COLOR_GRAY2BGR)

现在录制的视频应该可以正常打开了。请再次尝试录制:

1. 点击 "Connect" 连接相机
2. 点击 "Open" 打开相机
3. 点击 "Capture" 开启预览
4. 点击 "Record" 开始录制
   - 这次应该能在控制台看到图像尺寸和通道数信息
5. 等待几秒后再次点击 "Record" 停止录制

如果视频还是有问题,请告诉我控制台输出的图像尺寸和通道数信息,这样我可以进一步诊断问题。

------------------
# 正确代码，录制视频的按钮record-button


cpp
````
   void Pose_Kalman::recordVideo()
{
    Mat imageRead, tansImg;
    static int numFlag = 0;
    static bool isVideoWriterInitialized = false;
    static VideoWriter videoCreate;

    if(cameraOpen) {
        // 获取相机图像
        while(cameracontrol->grabImage(imageRead) == 0) {
            /*break;*/
        }

        if(!imageRead.empty()) {
            if(recordFlag == 1) {
                if(!isVideoWriterInitialized) {
                    // 获取实际图像尺寸和通道数
                    Size imgSize = imageRead.size();
                    int channels = imageRead.channels();
                    std::cout << "图像尺寸: " << imgSize.width << "x" << imgSize.height 
                              << " 通道数: " << channels << std::endl;
                    
                    // 创建视频写入器
                    QString savePath = videoSavePath.isEmpty() ? QString("./") : videoSavePath;
                    QString fileName = videoFileName.isEmpty() ? QString("recording.avi") : videoFileName;
                    QString fullPath = savePath + "/" + fileName;
                    
                    // 如果是灰度图像,需要转换为BGR
                    bool isColor = (channels == 3);
                    videoCreate.open(fullPath.toStdString(), 
                        VideoWriter::fourcc('M','J','P','G'), 25, imgSize, isColor);
                    
                    if(!videoCreate.isOpened()) {
                        std::cout << "无法创建视频文件!" << std::endl;
                        recordFlag = -1;
                        return;
                    }
                    isVideoWriterInitialized = true;
                    std::cout << "开始录制视频,保存至: " << fullPath.toStdString() << std::endl;
                }
                
                // 写入当前帧
                tansImg = imageRead.clone();
                if(imageRead.channels() == 1) {
                    // 如果是灰度图像,转换为BGR
                    cvtColor(tansImg, tansImg, COLOR_GRAY2BGR);
                }
                videoCreate.write(tansImg);
                numFlag++;
                std::cout << "已录制 " << numFlag << " 帧" << std::endl;
            } 
            else if(recordFlag == 0) {
                if(isVideoWriterInitialized) {
                    std::cout << "正在保存视频,共 " << numFlag << " 帧..." << std::endl;
                    videoCreate.release();
                    isVideoWriterInitialized = false;
                    std::cout << "视频保存完成!" << std::endl;
                    numFlag = 0;
                }
                recordFlag = -1;
            }
        } else {
            std::cout << "获取图像失败!" << std::endl;
        }
    }
}


````
======================
# 结果是图像的大小出现的问题
pose_kalman.h
````
  //yrrz 录制视频record 的成员变量名
     QString videoSavePath; // 视频保存路径
     QString videoFileName; // 视频文件名
     void setVideoSavePath(const QString &path) { videoSavePath = path; }
     void setVideoFileName(const QString &name) { videoFileName = name; }
     static VideoWriter* videoWriter;  // 视频写入器
     static int frameCount;  // 帧计数器

````
-------------------------
pose_kalman.cpp
````
// 初始化静态成员变量yrr
VideoWriter* Pose_Kalman::videoWriter = nullptr;
int Pose_Kalman::frameCount = 0;




void Pose_Kalman::recordVideo()
{
    Mat tansImg,trainimg;

    if(cameraOpen) {
        // 获取相机图像
        while(cameracontrol->grabImage(trainimg)==0) {
            /*break;*/
        }

        // 根据录制状态处理
        if(recordFlag == 1) {  // 开始录制
            if(videoWriter == nullptr) {
                QString savePath = videoSavePath.isEmpty() ? QString("./") : videoSavePath;
                QString fileName = videoFileName.isEmpty() ? QString("recording%d_%d.avi") : videoFileName;

                //sprintf(filename,"D:/QT_space/MicroSystem/image/07221/%d_%d.bmp",yNumber,iteratorNum);
                //static VideoWriter videoCreate("planeMove.avi", VideoWriter::fourcc('D', 'I', 'V', 'X'), 25, Size(1024,542),1);

                // 创建保存目录
                QDir dir(savePath);
                if(!dir.exists()) {
                    dir.mkpath(".");
                }

                QString fullPath = savePath + "/" + fileName;
                std::cout << "start record " << fullPath.toStdString() << std::endl;

                int channels =trainimg.channels();
                std::cout<<"pictures' channels:"<<channels<<endl;
                std::cout<<"pictures' size:"<<trainimg.size()<<endl;


                // 创建视频写入器
//                videoWriter = new VideoWriter(fullPath.toStdString(),
//                    VideoWriter::fourcc('D','I','V','X'), 25,Size(1024,542)/* Size(4096,2168),*/ ,true);
                  videoWriter = new VideoWriter(fullPath.toStdString(),
                      VideoWriter::fourcc('M','J','P','G'), 25,Size(1024,542)/* Size(4096,2168),*/ ,true);
//                videoWriter = new VideoWriter(fullPath.toStdString(),
//                    VideoWriter::fourcc('M','P','4','V'), 25, Size(4096,2168) ,true);
//                videoWriter = new VideoWriter(fullPath.toStdString(),
//                    VideoWriter::fourcc('X','2','6','4'), 25,Size(1024,542)/* Size(4096,2168),*/ ,true);


                if(!videoWriter->isOpened()) {
                    std::cout << "create faily!" << std::endl;
                    delete videoWriter;
                    videoWriter = nullptr;
                    recordFlag = -1;
                    return;
                }
                frameCount = 0;
            }

            // 写入当前帧
            tansImg = trainimg.clone();
            if(tansImg.channels()==1)
            {
                cvtColor(tansImg,tansImg,COLOR_GRAY2BGR);
            }
//            // 发送原始图像到UI显示
            Mat displayImg = trainimg.clone();
            if(displayImg.channels()==1)
            {
                cvtColor(displayImg,displayImg,COLOR_GRAY2BGR);
            }
            QImage showImg = MatToQImage(displayImg);
            emit sendImage(showImg);
            //调整大小写入帧
            if (tansImg.size() != Size(1024, 542)) {
                resize(tansImg, tansImg, Size(1024, 542));
            }
            videoWriter->write(tansImg);
            frameCount++;

            std::cout << "already record " << frameCount << " framecount" << std::endl;

//            // 实时显示当前帧，直接多了一个窗口
//            imshow("Recording", tansImg);  // 使用 OpenCV 的 imshow 显示画面
//            waitKey(1);  // 非阻塞等待，确保画面刷新
        } else if(recordFlag == 0) {  // 停止录制
            if(videoWriter != nullptr) {
                std::cout << "saving video now is " << frameCount << " framecount..." << std::endl;
                videoWriter->release();
                delete videoWriter;
                videoWriter = nullptr;
                std::cout << "save successfully" << std::endl;
                frameCount = 0;
            }
            recordFlag = -1;
        }
         //如果不在录制状态,仍然需要显示实时图像
        else if(recordFlag == -1) {
            Mat displayImg = trainimg.clone();
            if(displayImg.channels()==1)
            {
                cvtColor(displayImg,displayImg,COLOR_GRAY2BGR);
            }
            QImage showImg = MatToQImage(displayImg);
            emit sendImage(showImg);
        }
    }
}

````
---------------

mainwindow.cpp
````
void MainWindow::on_record_clicked()
{


    if(decision_task->imageCollect->recordFlag == -1) {  // 当前未在录制

        decision_task-> imageCollect->setVideoSavePath("E:/Yangranran/exercise/Microsystem/BUILD/videos");  // 设置保存路径
         decision_task->imageCollect->setVideoFileName("luship.avi");   // 设置文件名
        decision_task->imageCollect->recordFlag = 1;  // 开始录制
         decision_task->imageCollect->funSelect = 9;  // 使用 pose_kalman -case 9 进行录制
        ui->record->setText("Stop");
    } else {  // 正在录制
        decision_task->imageCollect->recordFlag = 0;  // 停止录制
        ui->record->setText("Record");
    }
}
````
------

mainwindow.cpp connect 的连接
````

      // 添加相机录制 record button 相关的连接 yrr
      connect(decision_task->imageCollect,&Pose_Kalman::sendImage,this,&MainWindow::showImage);
      connect(this,&MainWindow::on_record_clicked,[=](){
          if(decision_task->imageCollect->recordFlag == -1) {
              decision_task->imageCollect->funSelect = 8;  // 启动相机显示
          }
      });
````
