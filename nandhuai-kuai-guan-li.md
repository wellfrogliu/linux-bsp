
###nand flash 存储结构如图所示：
![](http://img.blog.csdn.net/20160425140639896?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
上图是一个nand flash 大小为128MB，其中1 page=2 Kbyte + 128 byte或64 byte，2K为数据区域，128 byte为oob区域。

**读写的最小操作单元为1 page，erase的最小单元为1 block**




参考文献：
1. MTD\(4\)---nand flash的bbt坏块表的建立函数代码分析[http://blog.csdn.net/zhanzheng520/article/details/11770359](http://blog.csdn.net/zhanzheng520/article/details/11770359)
2. 


