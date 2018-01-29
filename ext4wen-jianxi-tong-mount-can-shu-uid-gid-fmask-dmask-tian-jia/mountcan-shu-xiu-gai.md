#mount参数修改
&emsp;&emsp;在Linux文件系统中，ext2、ext3以及ext4默认的mount参数是不支持uid、gid、fmask、dmask等参数的，在某些场景下，例如硬盘中的文件是由root创建的，其他普通用户将无法修改硬盘中的文件，这对多用户的情况来说带来了很大的不变。需要解决的问题如何修改文件的权限，达到最优的结果。
方案：
1. 在mount后，使用chmod命令批量修改文件的权限，该方案的缺点是文件太多时，速度太慢；
2. 使用bindfs库，该库可以将一个目录挂载到其他目录，并且修改目录的权限。具体可以参考[https://bindfs.org/](https://bindfs.org/) 。但是该方案基于fuse，性能较差；
3. 修改ext4源代码，实现在mount的时候添加uid、gid、fmask、dmask等参数，完成文件权限的修改。

&emsp;&emsp;修改uid和giu的代码，可以参考Linux官方的diff文件修改。
[1-3-implement-uid-and-gid-mount-options-for-ext2.diff](/assets/1-3-implement-uid-and-gid-mount-options-for-ext2.diff)
[2-3-implement-uid-and-gid-mount-options-for-ext3.diff](/assets/2-3-implement-uid-and-gid-mount-options-for-ext3.diff)
[3-3-implement-uid-and-gid-mount-options-for-ext4.diff](/assets/3-3-implement-uid-and-gid-mount-options-for-ext4.diff)

**但是修改uid和gid只是修改了文件和文件夹的拥有者和组，满足了普通用户访问root创建的文件的需求，但是无法满足多个不同的普通用户同时访问的需求**

本文重点介绍fmask和dmask的修改，可以满足多用户的需求。
uid、gid、fmask、dmask等mount参数都在fs/ext4/ext4.h文件中的ext4_sb_info结构体中定义的。
```c
struct ext4_sb_info {
	unsigned long s_desc_size;	/* Size of a group descriptor in bytes */
	unsigned long s_inodes_per_block;/* Number of inodes per block */
	unsigned long s_blocks_per_group;/* Number of blocks in a group */
	unsigned long s_clusters_per_group; /* Number of clusters in a group */
	unsigned long s_inodes_per_group;/* Number of inodes in a group */
	unsigned long s_itb_per_group;	/* Number of inode table blocks per group */
	unsigned long s_gdb_count;	/* Number of group descriptor blocks */
	unsigned long s_desc_per_block;	/* Number of group descriptors per block */
	ext4_group_t s_groups_count;	/* Number of groups in the fs */
	ext4_group_t s_blockfile_groups;/* Groups acceptable for non-extent files */
	unsigned long s_overhead;  /* # of fs overhead clusters */
	unsigned int s_cluster_ratio;	/* Number of blocks per cluster */
	unsigned int s_cluster_bits;	/* log2 of s_cluster_ratio */
	loff_t s_bitmap_maxbytes;	/* max bytes for bitmap files */
	struct buffer_head * s_sbh;	/* Buffer containing the super block */
	struct ext4_super_block *s_es;	/* Pointer to the super block in the buffer */
	struct buffer_head **s_group_desc;
	unsigned int s_mount_opt;
	unsigned int s_mount_opt2;
	unsigned int s_mount_flags;
	unsigned int s_def_mount_opt;
	ext4_fsblk_t s_sb_block;
	atomic64_t s_resv_clusters;
	kuid_t s_resuid;
	kgid_t s_resgid;
	***
	unsigned short fs_fmask; 
	unsigned short fs_dmask;
	****
	unsigned short s_mount_state;
	unsigned short s_pad;
	int s_addr_per_block_bits;
	int s_desc_per_block_bits;
	int s_inode_size;
	int s_first_ino;
	unsigned int s_inode_readahead_blks;
	unsigned int s_inode_goal;
	spinlock_t s_next_gen_lock;
	u32 s_next_generation;
	u32 s_hash_seed[4];
	int s_def_hash_version;
	int s_hash_unsigned;	/* 3 if hash should be signed, 0 if not */
	struct percpu_counter s_freeclusters_counter;
	struct percpu_counter s_freeinodes_counter;
	struct percpu_counter s_dirs_counter;
	struct percpu_counter s_dirtyclusters_counter;
	struct blockgroup_lock *s_blockgroup_lock;
	struct proc_dir_entry *s_proc;
	struct kobject s_kobj;
	struct completion s_kobj_unregister;
	struct super_block *s_sb;
	...省略...
};
```
其中添加的代码如下：
```c
unsigned short fs_fmask;
unsigned short fs_dmask;
```
ext4_sb_info 结构体是ext4在内存中的超级块，记录了关于ext4的相关信息，该结构体并不会写入磁盘，仅仅存在于内存中。该结构体在ext4/super.c文件中的ext4_fill_super函数中初始化。该函数在mount的时候被调用，即该结构体在mount的时候才进行数据填充。由于我们没有在ext4_fill_super函数中对fs_fmask和fs_dmask进行初始化，所以这两个函数的默认值都是0（此处个人觉得应该初始化），那么fs_fmask和fs_dmask这两个变量是怎么赋值的呢？

&emsp;&emsp;fs_fmask和fs_dmask这两个变量主要通过mount的时候进行赋值，下面主要介绍一下赋值的流程：
在super.c文件中static const match_table_t tokens这个结构体中定义了mount的参数，如下：
```c	
static const match_table_t tokens = {
	...省略...
	{Opt_uid, "uid=%u"},
	{Opt_diskuid, "uid=%u:%u"},
	{Opt_gid, "gid=%u"},
	{Opt_diskgid, "gid=%u:%u"},
	{Opt_dmask, "dmask=%o"},
	{Opt_fmask, "fmask=%o"},
	{Opt_err, NULL},
	};
```
Opt_dmask和Opt_fmask在枚举变量enum中定义，如下：
```c
enum {
	Opt_bsd_df, Opt_minix_df, Opt_grpid, Opt_nogrpid,
	Opt_resgid, Opt_resuid, Opt_sb, Opt_err_cont, Opt_err_panic, Opt_err_ro,
	Opt_nouid32, Opt_debug, Opt_removed,
	Opt_user_xattr, Opt_nouser_xattr, Opt_acl, Opt_noacl,
	Opt_auto_da_alloc, Opt_noauto_da_alloc, Opt_noload,
	Opt_commit, Opt_min_batch_time, Opt_max_batch_time, Opt_journal_dev,
	Opt_journal_path, Opt_journal_checksum, Opt_journal_async_commit,
	Opt_abort, Opt_data_journal, Opt_data_ordered, Opt_data_writeback,
	Opt_data_err_abort, Opt_data_err_ignore, Opt_test_dummy_encryption,
	Opt_usrjquota, Opt_grpjquota, Opt_offusrjquota, Opt_offgrpjquota,
	Opt_jqfmt_vfsold, Opt_jqfmt_vfsv0, Opt_jqfmt_vfsv1, Opt_quota,
	Opt_noquota, Opt_barrier, Opt_nobarrier, Opt_err,
	Opt_usrquota, Opt_grpquota, Opt_i_version, Opt_dax,
	Opt_stripe, Opt_delalloc, Opt_nodelalloc, Opt_mblk_io_submit,
	Opt_lazytime, Opt_nolazytime,
	Opt_nomblk_io_submit, Opt_block_validity, Opt_noblock_validity,
	Opt_inode_readahead_blks, Opt_journal_ioprio,
	Opt_dioread_nolock, Opt_dioread_lock,
	Opt_discard, Opt_nodiscard, Opt_init_itable, Opt_noinit_itable,
	Opt_max_dir_size_kb, Opt_nojournal_checksum,
	Opt_uid, Opt_diskuid, Opt_gid, Opt_diskgid,Opt_fmask, Opt_dmask, 
};
```
通过上述过程就定义了mount的参数dmask和fmask，可以在mount的时候使用这些参数了。但是mount的参数dmask和fmask时如何与ext4_sb_info结构体中fs_fmask和fs_dmask联系起来的呢？

&emsp;&emsp;

函数parse_options主要负责对命令行进行数据解析，该函数调用了handle_mount_opt函数，来对fs_fmask和fs_dmask进行赋值，通过switch case结构来对token的值（match_table_t 结构体中定义）进行判断，从而实现数据的赋值，代码如下：
```c
switch (token) {
	case Opt_dmask:
		if (match_octal(&args[0], &arg)) {
			return -1;
		}
		sbi->fs_dmask = arg;
		return 1;	
	case Opt_fmask:
		if (match_octal(&args[0], &arg)) {
			return 0;
		}
		sbi->fs_fmask = arg;
		printk("fs_fmask:%d\n",sbi->fs_fmask);
		return 1;
```