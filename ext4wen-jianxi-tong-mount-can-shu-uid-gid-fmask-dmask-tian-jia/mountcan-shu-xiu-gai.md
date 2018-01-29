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