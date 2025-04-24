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
