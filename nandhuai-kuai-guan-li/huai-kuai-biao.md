&emsp;&emsp;坏块表简称BBT（bad block table），Linux kernel的bbt做的也比较简单，就是把整个flash的block在内存里面用2bit位图来标识good/bad，这样，在上层判断一个block是否good时就不需要再去读取flash的oob里面的坏块标记了，只需要读取内存里面的bbt就可以了，这是一个比较重要的优化。
&ensp;&ensp;&ensp;&ensp;坏块表一般是在boot或者kernel第一次运行时建立的，后面会存储在内存和nand的某些block上，一般为第一个或者最后几个block上。
&emsp;&emsp;每次Linux启动时，都会先查询nand中是否存在BBT block，如果存在，则直接将BBT读取至内存中，否则的话，则会创建BBT，并将新建的BBT写入nand中。

&emsp;&emsp;主要涉及到的文件有：

####BBT的扫描与新建
1. **spi_nand_mtd_register**：nand驱动、nand芯片识别以及nand相关成员函数的初始化，位于spi-nand-mtd.c文件中。
当加载nand驱动时，会首先调用spi_nand_mtd_register_函数，对nand驱动进行注册。该函数会对spi_nand_chip很_nand_chip的某些成员函数进行初始化，未初始化的函数留给后面的函数进行初始化。在该函数中会进行nand ID识别，通过调用nand_scan_ident函数实现。检测到nand芯片后，接着会调用nand_scan_tail函数，主要完成nand相关结构体 中未初始化成员函数的初始化以及BBT的扫描和新建；接着会调用nand_register函数对nand进行注册。

2. **nand_scan_tail**：位于nand_base.c文件中，主要完成nand相关结构体 中未初始化成员函数的初始化以及BBT的扫描和新建。在函数中


2. **nand_scan_bbt**：位于nand_bbt.c_文件中，主要完成扫描、查找、读取以及当bbt不存在时，创建bbt。重点关注search_read_bbts和check_create 两个函数。其中search_read_bbts函数主要完成bbt的扫描；check_create 主要完成在bbt不存在时，创建以及写入bbt。

3. **search_read_bbts**：主要完成主要完成bbt的扫描工作，通过调用search_bbt函数实现。

4. **search_bbt**：扫描设备的坏块表，读取nand上的块数据，与坏块表模板匹配，成功则记录坏块表的地址。
 - 判断读取block的方向，
 ```c
if (td->options & NAND_BBT_LASTBLOCK) {
		startblock = (mtd->size >> this->bbt_erase_shift) - 1;
		dir = -1;
	} else {
		startblock = 0;
		dir = 1;
	}
```

2. nand_base.c文件：
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
其中nand_scan_ident主要完成nand ID的识别以及跟chip相关的成员函数的初始化，nand_scan_tail主要完成