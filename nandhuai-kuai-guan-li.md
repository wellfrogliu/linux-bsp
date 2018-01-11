
###nand flash 存储结构如图所示：
![](http://img.blog.csdn.net/20160425140639896?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
上图是一个nand flash 大小为128MB，其中1 page=2 Kbyte + 128 byte或64 byte，2K为数据区域，128 byte为oob区域。

**读写的最小操作单元为1 page，erase的最小单元为1 block**
####坏块的定义：
坏块是指nand中不能进行读写或者擦除操作的区域。由于擦除的最小单元为1个block，因此坏块也是以一个block为最小单元。每个坏块的标志位于该block的第一个page oob区的前两个字节。判断一个坏块是否为坏块，就可以判断这两个字节是否为非0xff值，若为0xff,则无坏块。nand在出厂时都会出现坏块，会在对应的块中标记为坏块。



参考文献：
1. MTD\(4\)---nand flash的bbt坏块表的建立函数代码分析&emsp;[http://blog.csdn.net/zhanzheng520/article/details/11770359](http://blog.csdn.net/zhanzheng520/article/details/11770359)
2. 


