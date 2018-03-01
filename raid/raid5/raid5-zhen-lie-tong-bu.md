####raid5 阵列同步线程注册
&emsp;&emsp;在raid5.c文件中，在函数run（）中通过mddev->sync_thread = md_register_thread(md_do_sync, mddev,"reshape");实现了同步函数的注册。我们可以知道raid5的同步处理函数为md_do_sync。
####raid5阵列同步的触发点
&emsp;&emsp;raid5的主线程注册函数是raid5d，该函数主要做一些阵列状态检查，是否需要同步等。同步的检查主要通过md_check_recovery函数来完成同步检查。
*md_check_recovery*的功能主要包括：
1. 是否需要更新超级块，必要时更新超级块；
2. 检查同步线程是否正在运行，如果



mutex_trylock, mutex_lock区别：
mutex_trylock：尝试获得一个互斥锁，立即返回；当返回值为0时表示获得成功，否则表示获得失败；
mutex_lock：获得一个互斥锁，一直等待，直到成功；