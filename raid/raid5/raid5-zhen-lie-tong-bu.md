####raid5同步操作的调用流程
raid5d() -> md_check_recovery() -> md_do_sync() -> sync_request() -> handle_stripe()
其中
上述过程是一个同步操作过程的函数调用关系链。从这个关系链中可以知道，同步操作最后是由handle_stripe（）函数来实现的。
####raid5 阵列同步线程注册
&emsp;&emsp;在raid5.c文件中，在函数run（）中通过mddev->sync_thread = md_register_thread(md_do_sync, mddev,"reshape");实现了同步函数的注册。我们可以知道raid5的同步处理函数为md_do_sync。
####raid5阵列同步的触发点
&emsp;&emsp;raid5的主线程注册函数是raid5d，该函数主要做一些阵列状态检查，是否需要同步等。同步的检查主要通过md_check_recovery函数来完成同步检查。
**md_check_recovery**的功能主要包括：
1. 是否需要更新超级块，必要时更新超级块；
2. 检查同步线程是否正在运行，若正在运行，则不做任何处理；
3. 如果同步完成，则进行清理，并且将热备份盘激活；
4. 如果有坏盘，则移除该盘；
5. 如果阵列处于降级状态，则尝试添加热备盘；
6. 如果阵列有热备盘或者不处于同步状态，则启动同步线程。



mutex_trylock, mutex_lock区别：
mutex_trylock：尝试获得一个互斥锁，立即返回；当返回值为0时表示获得成功，否则表示获得失败；
mutex_lock：获得一个互斥锁，一直等待，直到成功；