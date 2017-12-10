##Linux spi框架简介
&emsp;&emsp;linux中的spi源码位于/drivers/spi目录下。spi框架主要包括3层，即spi核心层、spi控制器驱动层、spi设备驱动层。

- spi核心层位于spi.c中，主要提供spi的数据结构定义，spi控制器驱动以及spi设备驱动的注册与注销。与硬件无关，实现了spi message消息队列的管理，并且为异步spi实现了工作线程。为上层提供了同步spi和异步spi传输的方法。
- spi控制器驱动层位于spi-xx.c(spi-s3c64xx.c)中，与具体硬件有关，由用户完成具体spi协议的实现以及spi控制器寄存器的控制。
- spi设备层位于spi-dev.c中，实现spi控制器的操作函数在文件系统中注册，供用户空间操作spi是api。