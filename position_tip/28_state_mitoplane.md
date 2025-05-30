    case state_mitoplane:
        emit sendfocusposition(1);
        planeSelection=-1;
        break;


   connect(decision_task->pose_plane,&Pose_Plane::sendfocusposition,this,&MainWindow::focuspositionget);


   void MainWindow:: focuspositionget(int taskFlag)//0-初始平面 1-现实平面 2-最新平面位置
{
    if(taskFlag==0)
    {
        decision_task->patchClampX->focuspositionget(decision_task->pose_plane->focusinitplane);
    }
    else if (taskFlag==1)
    {
        decision_task->patchClampX->focuspositionget(decision_task->pose_plane->focusmitoplane);
    }
    else if(taskFlag==2)
    {
        decision_task->patchClampX->focuspositionget(decision_task->pose_plane->focuscurrentplane);
    }
    }

