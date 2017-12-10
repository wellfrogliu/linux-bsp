###spi控制器驱动分析
&emsp;&emsp;spi控制器驱动与具体硬件有关，主要完成配置spi控制器相关寄存器，完成spi的数据传输。本文以spi_s3c64xx.c为例进行分析。


####数据结构介绍
#####spi平台驱动结构体
&emsp;&emsp;该结构体在linux/platform_device.h注册，是通用的驱动注册结构体。
```c
static struct platform_driver s3c64xx_spi_driver = {
	.driver = {
		.name	= "s3c64xx-spi",
		.pm = &s3c64xx_spi_pm,
		.of_match_table = of_match_ptr(s3c64xx_spi_dt_match),
	},
	.probe = s3c64xx_spi_probe,
	.remove = s3c64xx_spi_remove,
	.id_table = s3c64xx_spi_driver_ids,
};
```

####spi驱动注册

