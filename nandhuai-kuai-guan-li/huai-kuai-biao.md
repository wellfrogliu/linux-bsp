坏块表简称BBT（bad block table），Linux kernel的bbt做的也比较简单，就是把整个flash的block在内存里面用2bit位图来标识good/bad，这样，在上层判断一个block是否good时就不需要再去读取flash的oob里面的坏块标记了，只需要读取内存里面的bbt就可以了，这是一个比较重要的优化。  
    坏块表一般是在boot或者kernel第一次运行时建立的，后面会存储在内存和nand的某些block上，一般为第一个或者最后几个block上。  
  每次Linux启动时，都会先查询nand中是否存在BBT block，如果存在，则直接将BBT读取至内存中，否则的话，则会创建BBT，并将新建的BBT写入nand中。

主要涉及到的文件有：

#### BBT的扫描与新建

1. **spi\_nand\_mtd\_register**：nand驱动、nand芯片识别以及nand相关成员函数的初始化，位于spi-nand-mtd.c文件中。  
   当加载nand驱动时，会首先调用spi_nand\_mtd\_register_函数，对nand驱动进行注册。该函数会对spi\_nand\_chip很\_nand\_chip的某些成员函数进行初始化，未初始化的函数留给后面的函数进行初始化。在该函数中会进行nand ID识别，通过调用nand\_scan\_ident函数实现。检测到nand芯片后，接着会调用nand\_scan\_tail函数，主要完成nand相关结构体 中未初始化成员函数的初始化以及BBT的扫描和新建；接着会调用nand\_register函数对nand进行注册。

2. **nand\_scan\_tail**：位于nand\_base.c文件中，主要完成nand相关结构体 中未初始化成员函数的初始化以及BBT的扫描和新建。在函数中

3. **nand\_scan\_bbt**：位于nand_bbt.c_文件中，主要完成扫描、查找、读取以及当bbt不存在时，创建bbt。重点关注search\_read\_bbts和check\_create 两个函数。其中search\_read\_bbts函数主要完成bbt的扫描；check\_create 主要完成在bbt不存在时，创建以及写入bbt。

4. **search\_read\_bbts**：主要完成主要完成bbt的扫描工作，通过调用search\_bbt函数实现。

5. **search\_bbt**：扫描设备的坏块表，读取nand上的块数据，与坏块表模板匹配，成功则记录坏块表的地址。

   * 判断bbt的查找方向，通过td-&gt;options & NAND\_BBT\_LASTBLOCK判断bbt的查找方向即block的读取方向。NAND\_BBT\_LASTBLOCK表示bbt查找方向为从最后向前，一般bbt的查找都是从后向前搜索。
     ```c
     if (td->options & NAND_BBT_LASTBLOCK) {
        startblock = (mtd->size >> this->bbt_erase_shift) - 1;
        dir = -1;
     } else {
        startblock = 0;
        dir = 1;
     }
     ```
   * 接着判断是否每个chip都包括一个bbt文件。一般每个chip都会包括一个bbt。代码如下：
     ```c
     /* Do we have a bbt per chip? */
     if (td->options & NAND_BBT_PERCHIP) {
        chips = this->numchips;
        bbtblocks = this->chipsize >> this->bbt_erase_shift;
        startblock &= bbtblocks - 1;
     } else {
        chips = 1;
        bbtblocks = mtd->size >> this->bbt_erase_shift;
     }
     ```
   * 读取block的非oob数据并与bbt关键字匹配。从后向前读取block中page数据，一共读取td-&gt;maxblocks次，该变量由NAND\_BBT\_SCAN\_MAXBLOCKS控制，由于nand可能存在最后四个block出现坏块的情况，因此可以适当将该值增大。

     ```c
     /* The maximum number of blocks to scan for a bbt */
     #define NAND_BBT_SCAN_MAXBLOCKS    4
     ```

     接着调用check\_pattern函数进行bbt block匹配，返回0，则表示找到了bbt，将该块的地址保存在td-&gt;pages\[i\]中，并且保存bbt的版本号。

6. **check\_pattern**：该函数主要完成检查传入的数据是否包含bbt模板，模板如下所示，如果传入洞房缓存区包含"Bbt0"则说明该block是bbt的block，如果包含“1tbB”则说明该block是bbt的镜像block。

   ```c
   /* Generic flash bbt descriptors */
   static uint8_t bbt_pattern[] = {'B', 'b', 't', '0' };
   static uint8_t mirror_pattern[] = {'1', 't', 'b', 'B' };
   ```

   至此search\_read\_bbts函数的功能已经分析完毕，下面继续分析check\_create函数。

7. **check\_create**：该函数主要根据上面的search\_read\_bbts扫描结果判断是否存在bbt，如果不存在，则在该函数进行新建，如果bbt丢失一个或者版本号不对，则更新该bbt。

   * 判断bbt是否存在。判断td-&gt;pages\[i\]是否为-1，判断是否存在bbt，如果td-&gt;pages\[i\] = -1， 则需要创建bbt，代码如下：  
     \`\`\`c

     ```c
     if (md) {   / Mirrored table available? /
            if (td->pages[i] == -1 && md->pages[i] == -1) {
                create = 1;
                writeops = 0x03;
            } else if (td->pages[i] == -1) {
                rd = md;
                writeops = 0x01;
            } else if (md->pages[i] == -1) {
                rd = td;
                writeops = 0x02;
            } else if (td->version[i] == md->version[i]) {
                rd = td;
                if (!(td->options & NAND_BBT_VERSION))
                    rd2 = md;
            } else if (((int8_t)(td->version[i] - md->version[i])) > 0) {
                rd = td;
                 writeops = 0x02;
             } else {
                 rd = md;
                 writeops = 0x01;
             } else {
                 if \(td-&gt;pages\[i\] == -1\) {  
                 create = 1;
                 writeops = 0x01;
             } else {
                 rd = td;
             }
     }
     ```

   * 这里面首先会判断md是否为0，即镜像bbt是否存在。

     * 如果create = 1，则需要创建bbt，调用create\_bbt函数完成创建工作，该函数后续再分析。创建代码如下：

       ```c
          /*Create the bad block table by scanning the device?*/ 
          if (create) { 
              if (!(td->options & NAND_BBT_CREATE)) 
                  continue;
              /* Create the table in memory by scanning the chip(s) */
              if (!(this->bbt_options & NAND_BBT_CREATE_EMPTY))
                  create_bbt(mtd, buf, bd, chipsel);
              td->version[i] = 1;
              if (md)
                  md->version[i] = 1;
          }
       ```

8. 如果不需要创建bbt，则rd = 1，会调用下面的代码：

   ```c
       if (rd) {
          res = read_abs_bbt(mtd, buf, rd, chipsel);
          if (mtd_is_eccerr(res)) {
              /* Mark table as invalid */
              rd->pages[i] = -1;
              rd->version[i] = 0;
              i--;
              continue;
          }
   ```

   在这段代码里面，会read\_abs\_bbt函数读取bbt的block数据，读取的数据长度与芯片的大小有关。然后对读取的数据做ecc校验，如果失败，则表明该block数据已经损坏，不能再用作bbt的block。

9. 接着会判断rd2是否为1，原理与rd=1的代码原理一样。主要跟md是否为1有关。

10. 接着如果需要创建bbt，则调用write\_bbt函数，写入bbt。代码如下：

    ```c
        if ((writeops & 0x01) && (td->options & NAND_BBT_WRITE)) {
           printf("write_bbt\n");
           printf("Write the bad block table to the device\n");
           res = write_bbt(mtd, buf, td, md, chipsel);
           if (res < 0)
               return res;
       }
    ```

    下面对write\_bbt函数进行分析。write\_bbt位于nand\_bbt.c文件中，主要完成bbt的写入（包括内存和nand对应的bbt block），首先循环查找可用于bbt区的nand blcok块，查找的

11. nand\_base.c文件：  
      在nand\_base.c文件中,主要完成nand的扫描与新建工作。首先调用nand\_scan函数进行nand扫描，

    ```c
    int nand_scan(struct mtd_info *mtd, int maxchips)
    {
     int ret;

     ret = nand_scan_ident(mtd, maxchips, NULL);
     if (!ret)
         ret = nand_scan_tail(mtd);
     return ret;
    }
    ```

    其中nand\_scan\_ident主要完成nand ID的识别以及跟chip相关的成员函数的初始化，nand\_scan\_tail主要完成



