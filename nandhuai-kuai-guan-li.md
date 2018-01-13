
###nand flash 存储结构如图所示：
![](http://img.blog.csdn.net/20160425140639896?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
上图是一个nand flash 大小为128MB，其中1 page=2 Kbyte + 128 byte或64 byte，2K为数据区域，128 byte为oob区域。

**读写的最小操作单元为1 page，erase的最小单元为1 block**
####坏块的定义：
坏块是指nand中不能进行读写或者擦除操作的区域。由于擦除的最小单元为1个block，因此坏块也是以一个block为最小单元。每个坏块的标志位于该block的第一个page oob区的前两个字节。判断一个坏块是否为坏块，就可以判断这两个字节是否为非0xff值，若为0xff,则无坏块。nand在出厂时都会出现坏块，会在对应的块中标记为坏块。

####ECC：
&emsp;&emsp;ECC的全称是Error Checking and Correction，nand flash 在读数据时有可能会出现一个或几个bit的错误，因此可以使用ECC进行纠正和校验。不同的nand的ECC校验能力不一样，ECC有1bit、4bit、8bit等，这些ECC位数一般指1bit/page、4bit/page、8bit/page等。

####OOB：
&emsp;&emsp;OOB(out of band)即spare area区，nand flash中每个page后都有一个oob区，主要用来存放硬件ECC校验码、坏块标记以及文件系统的组织信息，主要用于硬件纠错很坏块处理。一般一个page大小为2Kbyte,oob区一般为128byte或者64byte。
&emsp;&emsp;page可以分为大页和小页，目前主流的nand为大页。大页（big page）每个page大小为2Kbyte，坏块存储在每个block中第一个page中的第1和第2个字节中。

####BBT：
&emsp;&emsp;BBT(bad block table),不同厂家对坏块管理的方法不同，比如专门使用nand做存储的，会把bbt放到block0，因为**第0个block一定是好块**，但是如果nand被用来存储uboot，则第0个block不能用来存储BBT了。还有另一种做法是将BBT放在最后一个block中，但是必须保证该块不是坏块。BBT大小与nand的大小有关，nand越大，BBT就越大。通常系统启动时会查找BBT的位置，但是仅会查找maxblocks(linux driver定义在nand_bbt_descr结构体中)个block中，并且一般是从最后一个block中查找。BBT一般都有一个BBT备份block。
&emsp;&emsp;在BBT中每2个bit表示一个block。

nand中与bbt的有关函数位于\drivers\mtd\spi-nand\spi-nand-bbt.c中（注意spi-nand-bbt.c是spi nand）。
参考文献：
1. MTD\(4\)---nand flash的bbt坏块表的建立函数代码分析&emsp;[http://blog.csdn.net/zhanzheng520/article/details/11770359](http://blog.csdn.net/zhanzheng520/article/details/11770359)
2. nand flash 坏块&emsp;[http://blog.csdn.net/seasonyrq/article/details/51510965](http://blog.csdn.net/seasonyrq/article/details/51510965)


