&emsp;&emsp;坏块表简称BBT（bad block table），Linux kernel的bbt做的也比较简单，就是把整个flash的block在内存里面用2bit位图来标识good/bad，这样，在上层判断一个block是否good时就不需要再去读取flash的oob里面的坏块标记了，只需要读取内存里面的bbt就可以了，这是一个比较重要的优化。
&ensp;&ensp;&ensp;&ensp;坏块表一般是在boot或者kernel第一次运行时建立的，后面会存储在内存和nand的某些block上，一般为第一个或者最后几个block上。
&emsp;&emsp;每次Linux启动时，都会先查询nand中是否存在BBT block，如果存在，则直接将BBT读取至内存中，否则的话，则会创建BBT，并将新建的BBT写入nand中。

&emsp;&emsp;主要涉及到的文件有：

1. nand_base.c文件：
&emsp;&emsp;在nand_base.c文件中,主要完成nand的扫描与新建工作。首先调用nand_scan函数进行nand扫描，int nand_scan(struct 
```c
mtd_info *mtd, int maxchips)
{
	int ret;

	ret = nand_scan_ident(mtd, maxchips, NULL);
	if (!ret)
		ret = nand_scan_tail(mtd);
	return ret;
}
```