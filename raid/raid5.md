###raid 5的线程主要包括主线程和同步线程
1. 主线程注册是在setup_conf函数中完成的，在run函数中通过mddev->thread = conf->thread; 将主线程句柄保存在mddev中；（raid5.c）
2. 阵列同步线程注册是在run函数中，通过mddev->sync_thread = md_register_thread(md_do_sync, mddev,"reshape");完成注册。