#mount参数修改
在Linux文件系统中，ext2、ext3以及ext4默认的mount参数是不支持uid、gid、fmask、dmask等参数的，在某些场景下，例如硬盘中的文件是由root创建的，其他普通用户将无法修改硬盘中的文件，这对多用户的情况来说带来了很大的不变。需要解决的问题是在mount的时候添加uid、gid、fmask、dmask等参数来修改文件的权限。
方案：
1. 在mount后，使用chmod命令批量修改文件的权限，该方案的缺点是文件太多时，速度太慢；
2. 使用bindfs库，该库可以将一个目录挂载到其他目录，并且修改目录的权限。具体可以参考[](https://bindfs.org/) 。但是该方案基于fuse，性能较差。