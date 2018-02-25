####raid5 阵列同步线程注册
&emsp;&emsp;在raid5.c文件中，在函数run（）中通过mddev->sync_thread = md_register_thread(md_do_sync, mddev,"reshape");实现了同步函数的注册。我们可以知道raid5的同步处理函数为md_do_sync。
####raid5阵列同步的触发点
&emsp;&emsp;



mutex_trylock, mutex_lock区别：
mutex_trylock：尝试获得一个互斥锁，立即返回；当返回值为0时表示获得成功，否则表示获得失败；
mutex_lock：获得一个互斥锁，一直等待，直到成功；