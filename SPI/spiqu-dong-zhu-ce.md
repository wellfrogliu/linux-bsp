###spi驱动注册与销毁流程分析
&emsp;&emsp;spi驱动注册与销毁主要包括spi控制器驱动以及spi设备驱动的注册与销毁。
####spi子系统的注册
&emsp;&emsp;spi控制器注册函数spi_init位于spi.c文件中，函数实现如下所示：
```c
static int __init spi_init(void)
{
	int	status;
	buf = kmalloc(SPI_BUFSIZ, GFP_KERNEL);
	if (!buf) {
		status = -ENOMEM;
		goto err0;
	}
	status = bus_register(&spi_bus_type);
	if (status < 0)
		goto err1;
	status = class_register(&spi_master_class);
	if (status < 0)
		goto err2;
	if (IS_ENABLED(CONFIG_OF_DYNAMIC))
		WARN_ON(of_reconfig_notifier_register(&spi_of_notifier));					   			    
	return 0;
	
err2:
	bus_unregister(&spi_bus_type);
err1:
	kfree(buf);
	buf = NULL;
err0:
	return status;
}
```
&emsp;&emsp;在该函数中，首先进行spi总线的注册，接着进行spi控制器类的注册，分别对应/sys/bus/下的spi目录和/sys/class/下的spi_master目录。
如果注册失败，则调用bus_unregister清除spi总线驱动以及释放开辟的空间。

####spi控制器驱动的注册

####spi控制器注册






