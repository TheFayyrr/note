    case state_lensdown:
        planeSelection=lensmove(-1);//对焦点击驱动向下移动30微米
        break;



  int8_t Pose_Plane:: lensmove(int direction)
{
    qDebug()<<"send lens and scanning move is running "<<endl;
    if (direction>0)
    {
        qDebug()<<" "<<endl;
        qDebug()<<"lens move  up   30 microns "<<endl;
    }
    else {
        {
            qDebug()<<" "<<endl;
            qDebug()<<"lens move down  30 microns "<<endl;
        }
    }
    emit sendlens(300,direction,0);//驱动物镜上升30micron

    return state_idle;
}
