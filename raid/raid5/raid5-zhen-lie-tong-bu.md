####raid5同步操作的调用流程
&emsp;&emsp;raid5d() -> md_check_recovery() -> md_do_sync() -> sync_request() -> handle_stripe()
&emsp;&emsp;上述过程是一个同步操作过程的函数调用关系链。从这个关系链中可以知道，同步操作最后是由handle_stripe（）函数来实现的。
&emsp;&emsp;其中raid5d()为raid5的守护线程，md_check_recovery() 主要进行阵列状态检查，是否需要进行同步等操作；md_do_sync()主要为raid的同步处理线程，在该函数中会调用对应级别的raid同步函数sync_request()进行同步工作。下面对这些函数进行具体分析。

####raid5守护线程raid5d()
&emsp;&emsp;raid5d()为raid5的守护线程，其注册流程为：
md_run（）-> (pers->run(mddev)) -> setup_conf(mddev) -> md_register_thread(raid5d, mddev, pers_name)。
raid5d()函数调用md_check_recovery(mddev)函数进行检查，当需要同步时，启动同步线程进行同步。

####md_check_recovery()
&emsp;&emsp;同步的检查主要通过md_check_recovery函数来完成同步检查。
**md_check_recovery**的功能主要包括：
1. 是否需要更新超级块，必要时更新超级块；
2. 检查同步线程是否正在运行，若正在运行，则不做任何处理；
3. 如果同步完成，则进行清理，并且将热备份盘激活；
4. 如果有坏盘，则移除该盘；
5. 如果阵列处于降级状态，则尝试添加热备盘；
6. 如果阵列有热备盘或者不处于同步状态，则启动同步线程。

对应上面功能的代码实现如下：
```c
	if (mddev->flags & MD_UPDATE_SB_FLAGS) //根据标志位判断需要更新超级块
            md_update_sb(mddev, 0);//更新超级块
```
```c
void md_check_recovery(struct mddev *mddev)
{
	if (mddev->suspended) //阵列挂起
		return;

	if (mddev->bitmap) //bitmap清理工作（待确认）
		bitmap_daemon_work(mddev);

	if (signal_pending(current)) {//接收到进入safemode的信号,
		if (mddev->pers->sync_request && !mddev->external) {
			printk(KERN_INFO "md: %s in immediate safe mode\n",
			       mdname(mddev));
			mddev->safemode = 2;//将safemode赋值为2
		}
		flush_signals(current);//刷新主线程所有的待处理信号
	}

	if (mddev->ro && !test_bit(MD_RECOVERY_NEEDED, &mddev->recovery))//1. 该阵列只读 2. 未设置检查标志
		return;
	if ( ! (
		(mddev->flags & MD_UPDATE_SB_FLAGS & ~ (1<<MD_CHANGE_PENDING)) ||
		test_bit(MD_RECOVERY_NEEDED, &mddev->recovery) ||//检测是否设置了需要检查标志
		test_bit(MD_RECOVERY_DONE, &mddev->recovery) || //同步完成
		(mddev->external == 0 && mddev->safemode == 1) || //metadata位于磁盘内部，阵列处于安全模式
		(mddev->safemode == 2 && ! atomic_read(&mddev->writes_pending)
		 && !mddev->in_sync && mddev->recovery_cp == MaxSector)
		))//safemode为2有两种情况，一是系统重启时，二是7768行接收到信号时，第一种情况时in_sync为1，第二种情况可以触发更新超级块，根据in_sync标志写回磁盘resync_offset等等。
		return;

	if (mddev_trylock(mddev)) {//对mddev加锁，此处使用trylock可以避免主线程阻塞；而使用lock会导致主线程休眠
		int spares = 0;

		if (mddev->ro) {//阵列只读
			struct md_rdev *rdev;
			if (!mddev->external && mddev->in_sync)
				/* 'Blocked' flag not needed as failed devices
				 * will be recorded if array switched to read/write.
				 * Leaving it set will prevent the device
				 * from being removed.
				 */
				rdev_for_each(rdev, mddev)
					clear_bit(Blocked, &rdev->flags);//清除Blocked标志位（当设置Blocked后，该设备不会被移除）
			/* On a read-only array we can:
			 * - remove failed devices
			 * - add already-in_sync devices if the array itself
			 *   is in-sync.
			 * As we only add devices that are already in-sync,
			 * we can activate the spares immediately.
			 */
			remove_and_add_spares(mddev, NULL); //移除并添加热备盘
			/* There is no thread, but we need to call
			 * ->spare_active and clear saved_raid_disk
			 */
			set_bit(MD_RECOVERY_INTR, &mddev->recovery);
			md_reap_sync_thread(mddev);
			clear_bit(MD_RECOVERY_RECOVER, &mddev->recovery);
			clear_bit(MD_RECOVERY_NEEDED, &mddev->recovery);
			clear_bit(MD_CHANGE_PENDING, &mddev->flags);
			goto unlock;
		}

		if (!mddev->external) {//元数据位于阵列内部
			int did_change = 0;
			spin_lock(&mddev->lock);
			if (mddev->safemode &&//待理解？？
			    !atomic_read(&mddev->writes_pending) &&
			    !mddev->in_sync &&
			    mddev->recovery_cp == MaxSector) {
				mddev->in_sync = 1;
				did_change = 1;
				set_bit(MD_CHANGE_CLEAN, &mddev->flags);
			}
			if (mddev->safemode == 1)
				mddev->safemode = 0;//safemode为0 时，需要在同步完成后，设置safemode为1
			spin_unlock(&mddev->lock);
			if (did_change)
				sysfs_notify_dirent_safe(mddev->sysfs_state);//更新mddev的sysfs下状态。推测此处应该是阵列的状态信息更新到用户态
		}

		if (mddev->flags & MD_UPDATE_SB_FLAGS) //根据标志位判断需要更新超级块
			md_update_sb(mddev, 0);//更新超级块

		if (test_bit(MD_RECOVERY_RUNNING, &mddev->recovery) &&
		    !test_bit(MD_RECOVERY_DONE, &mddev->recovery)) {//同步线程正在工作，不需要再开始同步工作
			/* resync/recovery still happening */
			clear_bit(MD_RECOVERY_NEEDED, &mddev->recovery);
			goto unlock;
		}
		if (mddev->sync_thread) { //同步完成，获得结果，释放同步线程
			md_reap_sync_thread(mddev);
			goto unlock;
		}
		/* Set RUNNING before clearing NEEDED to avoid
		 * any transients in the value of "sync_action".
		 */
		mddev->curr_resync_completed = 0;
		spin_lock(&mddev->lock);
		set_bit(MD_RECOVERY_RUNNING, &mddev->recovery);
		spin_unlock(&mddev->lock);
		/* Clear some bits that don't mean anything, but
		 * might be left set
		 */
		clear_bit(MD_RECOVERY_INTR, &mddev->recovery);
		clear_bit(MD_RECOVERY_DONE, &mddev->recovery);

		if (!test_and_clear_bit(MD_RECOVERY_NEEDED, &mddev->recovery) ||
		    test_bit(MD_RECOVERY_FROZEN, &mddev->recovery))
			goto not_running;
		/* no recovery is running.
		 * remove any failed drives, then
		 * add spares if possible.
		 * Spares are also removed and re-added, to allow
		 * the personality to fail the re-add.
		 */

		if (mddev->reshape_position != MaxSector) {
			if (mddev->pers->check_reshape == NULL ||
			    mddev->pers->check_reshape(mddev) != 0)
				/* Cannot proceed */
				goto not_running;
			set_bit(MD_RECOVERY_RESHAPE, &mddev->recovery);
			clear_bit(MD_RECOVERY_RECOVER, &mddev->recovery);
		} else if ((spares = remove_and_add_spares(mddev, NULL))) {//启动同步任务
			clear_bit(MD_RECOVERY_SYNC, &mddev->recovery);
			clear_bit(MD_RECOVERY_CHECK, &mddev->recovery);
			clear_bit(MD_RECOVERY_REQUESTED, &mddev->recovery);
			set_bit(MD_RECOVERY_RECOVER, &mddev->recovery);
		} else if (mddev->recovery_cp < MaxSector) { //启动重建任务
			set_bit(MD_RECOVERY_SYNC, &mddev->recovery);
			clear_bit(MD_RECOVERY_RECOVER, &mddev->recovery);
		} else if (!test_bit(MD_RECOVERY_SYNC, &mddev->recovery))
			/* nothing to be done ... */
			goto not_running;

		if (mddev->pers->sync_request) {
			if (spares) {
				/* We are adding a device or devices to an array
				 * which has the bitmap stored on all devices.
				 * So make sure all bitmap pages get written
				 */
				bitmap_write_all(mddev->bitmap);
			}
			INIT_WORK(&mddev->del_work, md_start_sync);
			queue_work(md_misc_wq, &mddev->del_work);
			goto unlock;
		}
	not_running:
		if (!mddev->sync_thread) {
			clear_bit(MD_RECOVERY_RUNNING, &mddev->recovery);
			wake_up(&resync_wait);
			if (test_and_clear_bit(MD_RECOVERY_RECOVER,
					       &mddev->recovery))
				if (mddev->sysfs_action)
					sysfs_notify_dirent_safe(mddev->sysfs_action);
		}
	unlock:
		wake_up(&mddev->sb_wait);
		mddev_unlock(mddev);
	}
}
```

####md_do_sync()


####sync_request()


####handle_stripe()








####raid5 阵列同步线程注册
&emsp;&emsp;在raid5.c文件中，在函数run（）中通过mddev->sync_thread = md_register_thread(md_do_sync, mddev,"reshape");实现了同步函数的注册。我们可以知道raid5的同步处理函数为md_do_sync。
####raid5阵列同步的触发点






mutex_trylock, mutex_lock区别：
mutex_trylock：尝试获得一个互斥锁，立即返回；当返回值为0时表示获得成功，否则表示获得失败；
mutex_lock：获得一个互斥锁，一直等待，直到成功；