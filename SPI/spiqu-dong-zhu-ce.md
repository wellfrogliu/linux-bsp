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
&emsp;&emsp;spi控制器的注册与具体硬件有关，不同的硬件，驱动略有差异，但是都遵循Linux spi框架。本文以s3c64xx为例进行分析。
&emsp;&emsp;函数s3c64xx_spi_probe是s3c64xx控制器的注册函数。
```c
static int s3c64xx_spi_probe(struct platform_device *pdev)
{
	struct resource	*mem_res;
	struct s3c64xx_spi_driver_data *sdd;
	struct s3c64xx_spi_info *sci = dev_get_platdata(&pdev->dev);
	struct spi_master *master;
	int ret, irq;
	char clk_name[16];

	if (!sci && pdev->dev.of_node) {
		sci = s3c64xx_spi_parse_dt(&pdev->dev);
		if (IS_ERR(sci))
			return PTR_ERR(sci);
	}

	if (!sci) {
		dev_err(&pdev->dev, "platform_data missing!\n");
		return -ENODEV;
	}

	mem_res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
	if (mem_res == NULL) {
		dev_err(&pdev->dev, "Unable to get SPI MEM resource\n");
		return -ENXIO;
	}

	irq = platform_get_irq(pdev, 0);
	if (irq < 0) {
		dev_warn(&pdev->dev, "Failed to get IRQ: %d\n", irq);
		return irq;
	}

	master = spi_alloc_master(&pdev->dev,
				sizeof(struct s3c64xx_spi_driver_data));
	if (master == NULL) {
		dev_err(&pdev->dev, "Unable to allocate SPI Master\n");
		return -ENOMEM;
	}

	platform_set_drvdata(pdev, master);

	sdd = spi_master_get_devdata(master);
	sdd->port_conf = s3c64xx_spi_get_port_config(pdev);
	sdd->master = master;
	sdd->cntrlr_info = sci;
	sdd->pdev = pdev;
	sdd->sfr_start = mem_res->start;
	if (pdev->dev.of_node) {
		ret = of_alias_get_id(pdev->dev.of_node, "spi");
		if (ret < 0) {
			dev_err(&pdev->dev, "failed to get alias id, errno %d\n",
				ret);
			goto err0;
		}
		sdd->port_id = ret;
	} else {
		sdd->port_id = pdev->id;
	}

	sdd->cur_bpw = 8;

	if (!sdd->pdev->dev.of_node && (!sci->dma_tx || !sci->dma_rx)) {
		dev_warn(&pdev->dev, "Unable to get SPI tx/rx DMA data. Switching to poll mode\n");
		sdd->port_conf->quirks = S3C64XX_SPI_QUIRK_POLL;
	}

	sdd->tx_dma.direction = DMA_MEM_TO_DEV;
	sdd->rx_dma.direction = DMA_DEV_TO_MEM;

	master->dev.of_node = pdev->dev.of_node;
	master->bus_num = sdd->port_id;
	master->setup = s3c64xx_spi_setup;
	master->cleanup = s3c64xx_spi_cleanup;
	master->prepare_transfer_hardware = s3c64xx_spi_prepare_transfer;
	master->prepare_message = s3c64xx_spi_prepare_message;
	master->transfer_one = s3c64xx_spi_transfer_one;
	master->unprepare_transfer_hardware = s3c64xx_spi_unprepare_transfer;
	master->num_chipselect = sci->num_cs;
	master->dma_alignment = 8;
	master->bits_per_word_mask = SPI_BPW_MASK(32) | SPI_BPW_MASK(16) |
					SPI_BPW_MASK(8);
	/* the spi->mode bits understood by this driver: */
	master->mode_bits = SPI_CPOL | SPI_CPHA | SPI_CS_HIGH;
	master->auto_runtime_pm = true;
	if (!is_polling(sdd))
		master->can_dma = s3c64xx_spi_can_dma;

	sdd->regs = devm_ioremap_resource(&pdev->dev, mem_res);
	if (IS_ERR(sdd->regs)) {
		ret = PTR_ERR(sdd->regs);
		goto err0;
	}

	if (sci->cfg_gpio && sci->cfg_gpio()) {
		dev_err(&pdev->dev, "Unable to config gpio\n");
		ret = -EBUSY;
		goto err0;
	}

	/* Setup clocks */
	sdd->clk = devm_clk_get(&pdev->dev, "spi");
	if (IS_ERR(sdd->clk)) {
		dev_err(&pdev->dev, "Unable to acquire clock 'spi'\n");
		ret = PTR_ERR(sdd->clk);
		goto err0;
	}

	if (clk_prepare_enable(sdd->clk)) {
		dev_err(&pdev->dev, "Couldn't enable clock 'spi'\n");
		ret = -EBUSY;
		goto err0;
	}

	sprintf(clk_name, "spi_busclk%d", sci->src_clk_nr);
	sdd->src_clk = devm_clk_get(&pdev->dev, clk_name);
	if (IS_ERR(sdd->src_clk)) {
		dev_err(&pdev->dev,
			"Unable to acquire clock '%s'\n", clk_name);
		ret = PTR_ERR(sdd->src_clk);
		goto err2;
	}

	if (clk_prepare_enable(sdd->src_clk)) {
		dev_err(&pdev->dev, "Couldn't enable clock '%s'\n", clk_name);
		ret = -EBUSY;
		goto err2;
	}

	pm_runtime_set_autosuspend_delay(&pdev->dev, AUTOSUSPEND_TIMEOUT);
	pm_runtime_use_autosuspend(&pdev->dev);
	pm_runtime_set_active(&pdev->dev);
	pm_runtime_enable(&pdev->dev);
	pm_runtime_get_sync(&pdev->dev);

	/* Setup Deufult Mode */
	s3c64xx_spi_hwinit(sdd, sdd->port_id);

	spin_lock_init(&sdd->lock);
	init_completion(&sdd->xfer_completion);

	ret = devm_request_irq(&pdev->dev, irq, s3c64xx_spi_irq, 0,
				"spi-s3c64xx", sdd);
	if (ret != 0) {
		dev_err(&pdev->dev, "Failed to request IRQ %d: %d\n",
			irq, ret);
		goto err3;
	}

	writel(S3C64XX_SPI_INT_RX_OVERRUN_EN | S3C64XX_SPI_INT_RX_UNDERRUN_EN |
	       S3C64XX_SPI_INT_TX_OVERRUN_EN | S3C64XX_SPI_INT_TX_UNDERRUN_EN,
	       sdd->regs + S3C64XX_SPI_INT_EN);

	ret = devm_spi_register_master(&pdev->dev, master);
	if (ret != 0) {
		dev_err(&pdev->dev, "cannot register SPI master: %d\n", ret);
		goto err3;
	}

	dev_dbg(&pdev->dev, "Samsung SoC SPI Driver loaded for Bus SPI-%d with %d Slaves attached\n",
					sdd->port_id, master->num_chipselect);
	dev_dbg(&pdev->dev, "\tIOmem=[%pR]\tFIFO %dbytes\tDMA=[Rx-%p, Tx-%p]\n",
					mem_res, (FIFO_LVL_MASK(sdd) >> 1) + 1,
					sci->dma_rx, sci->dma_tx);

	pm_runtime_mark_last_busy(&pdev->dev);
	pm_runtime_put_autosuspend(&pdev->dev);

	return 0;

err3:
	pm_runtime_put_noidle(&pdev->dev);
	pm_runtime_disable(&pdev->dev);
	pm_runtime_set_suspended(&pdev->dev);

	clk_disable_unprepare(sdd->src_clk);
err2:
	clk_disable_unprepare(sdd->clk);
err0:
	spi_master_put(master);

	return ret;
}
```
####spi控制器注册





