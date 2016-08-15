---
layout: post
title:  MIPI to RGB
categories: 工作随笔
date: 2016-08-15 9:40:30
---

***

## 简介

屏的接口种类非常多，常见的包括 RGB、HDMI、VGA、LVDS、EDP、MIPI等接口。
其中，在 Android 移动设备上，大多采用的是MIPI接口。
某些时候，由于某种需求，需要将 Android 设备上的 MIPI 数据显示到其他接口的屏上，此时，则需要利用相关转换芯片将 MIPI 接口的数据转换成其他接口的数据。
在 msm8916 上有过这样的需求：将 MIPI 数据转换成 RGB 进行输出，当时采用的转换芯片是 ICN6211。以下记录之。

### RGB 简介

R(红)、G(绿)、B(蓝)是自然界三原色，三原色按照不同比例进行相加，混合成不同的颜色。
RGB其取值范围分别为0~255，其值越大，则越亮，当RGB都取值为255时，则为白色。相反全为0时，则为黑色。
而RGB接口就是分三原色进行输入的图形视频接口，也叫作色差分量接口。

RGB 接口中包括三大类信号：数据信号、时钟信号和控制信号。

+ 数据信号，通过数据线传输，一般分为6 bit 或者 8 bit传输(R0~R5/G0~G5/B0~B5 或 R0~R7/G0~G7/B0~B7)，也即是数据信号线分为18根或者24根

+ 时钟信号，特指像素时钟信号，传输数据和对数据信号进行读取的基准。

+ 控制信号，包括数据使能信号DE(或有效显示数据选通信号)、行同步信号HS、帧同步信号VS


### ICN6211 简介

ICN6211 是一款 MIPI 转 RGB 的芯片，下面是它的简要特性:

+ 其最高支持 MIPI DSI 的4 Lane 输入；当4lane输入时，其支持的最大带宽为1GBps

+ 其能对 MIPI DSI的 RGB565-16bpp、RGB666-18bpp 以及 RGB888-24bpp 的数据包进行解码转换

+ 其能输出 18 bpp 或者 24 bpp 像素的 RGB 图形数据，像素时钟的范围在 5MHz ~ 154MHz

+ 其支持的最大分辨率为 FHD(1920x1080p) 以及 WUXGA(1920x1200p)

+ 支持 I2C 接口

![](/assets/icn6211/icn6211_functional.png)

> icn6211 Functional Block Diagram


![](/assets/icn6211/icn6211_app.png)

> System Application Diagram

屏的显示分为两部分，一部分是开机显示厂商 logo，一部分是系统启动完毕后正常的显示。下面就这两部分的开发细节进行介绍。


## 添加 MIPI 转 RGB 支持

针对 MIPI 接口的屏，屏的配置参数通过 DATA0-、DATA0+ 这对数据线下发到屏端。
如果初始化正确，则切换到 HS 模式下，直接把数据从DATA0~DATA3刷出去即可。
那么针对 ICN6211-MIPI-RGB 的屏，又是怎么操作的呢？其实也非常简单，在给定 ICN6211 正确的电压后，将屏的配置参数通过 ICN6211 的I2C下发到屏端即可。
后续数据的传输则不再需要操心。

### ICN6211 参数图

![](/assets/icn6211/icn6211_para.png)

### bootloader 

在 `mdss_dsi_panel_initialize()` 函数中，会下发屏的配置参数。
它原本是调用 MIPI 接口函数 `mipi_dsi_cmds_tx() ` 去下发的参数，在这里改成或者添加 I2C 的传输接口即可。

	int mdss_dsi_panel_initialize(struct mipi_dsi_panel_config *pinfo, uint32_t
			broadcast)
	{
		if (pinfo->i2c_cmds_table) {
			mipi_i2c_cmds_tx(pinfo->i2c_cmds_table);
		}
	}


	static struct qup_i2c_dev  *i2c_dev;
	static int qrd_lcd_i2c_write(int addr, int reg, int val)
	{
		int ret = 0;
		uint8_t data_buf[] = { reg, val };
	
		/* Create a i2c_msg buffer, that is used to put the controller into write
		   mode and then to write some data. */
		struct i2c_msg msg_buf[] = { {addr,
			I2C_M_WR, 2, data_buf}
		};
	
		ret = qup_i2c_xfer(i2c_dev, msg_buf, 1);
		if(ret < 0) {
			dprintf(CRITICAL, "qup_i2c_xfer error %d\n", ret);
			return ret;
		}
		return 0;
	}

	/*write i2c command*/
	
	int mipi_i2c_cmds_tx(struct i2c_on_command_table *i2c_cmds_table)
	{
		int i = 0;
		i2c_dev = qup_blsp_i2c_init(BLSP_ID_1, QUP_ID_1, 100000, 19200000);
		if(!i2c_dev) {
			dprintf(CRITICAL, "qup_blsp_i2c_init failed \n");
			ASSERT(0);
		}
	
		for(i = 0; i < i2c_cmds_table->cmds_cnt; i++)
		{
			qrd_lcd_i2c_write(i2c_cmds_table->slave_addr, i2c_cmds_table->cmds_list[i].reg, i2c_cmds_table->cmds_list[i].val);
		}
	}


当然，` pinfo->i2c_cmds_table ` 这个table,按照 qcom 的屏处理流程，它是在 `msm8916/oem_panel.c` 里面进行处理的，下面直接贴出处理代码片段：

	case ICN6211_QHD_VIDEO_PANEL:
		panelstruct->paneldata    = &icn6211_qhd_video_panel_data;
		panelstruct->panelres     = &icn6211_qhd_video_panel_res;
		panelstruct->color        = &icn6211_qhd_video_color;
		panelstruct->videopanel   = &icn6211_qhd_video_video_panel;
		panelstruct->commandpanel = &icn6211_qhd_video_command_panel;
		panelstruct->state        = &icn6211_qhd_video_state;
		panelstruct->laneconfig   = &icn6211_qhd_video_lane_config;
		panelstruct->paneltiminginfo
				= &icn6211_qhd_video_timing_info;
		panelstruct->panelresetseq
				= &hx8389b_qhd_video_reset_seq;
		pinfo->mipi.i2c_cmds_table = &icn6211_i2c_on_command_table; 
		pinfo->mipi.i2c_cmds_table->cmds_cnt = ICN6211_QHD_VIDEO_ON_COMMAND;
		pinfo->mipi.i2c_cmds_table->slave_addr = ICN6211_I2C_SLAVE_ADDR;
		pinfo->mipi.i2c_cmds_table->blsp_id = I2C_BLSP_ID;
		pinfo->mipi.i2c_cmds_table->qup_id = I2C_QUP_ID;
		panelstruct->backlightinfo = &icn6211_qhd_video_backlight;
		memcpy(phy_db->timing,
				icn6211_qhd_video_timings, TIMING_SIZE);
		break;


### kernel

在 msm8916 的 kernel 部分，对一个屏驱动的处理分为三步：

+ 解析该屏的设备树文件(DTSI)

+ 注册屏初始化命令传输接口

+ 下发初始化命令，使能 Panel

下面就一步一步的来看怎么添加的:

1. 解析 I2C 命令

		static int mdss_panel_parse_dt(struct device_node *np,
				struct mdss_dsi_ctrl_pdata *ctrl_pdata)
		{
			//...
			rc = of_property_read_string(np,
					"qcom,mdss-command-access", &data);
				
				if (!rc && !strcmp(data, "i2c")) {  /*the property is self defined, it may not exist in some dts configure*/                                          
					ctrl_pdata->cmd_access = CMD_ACCESS_I2C;
						
						ctrl_pdata->i2c_handle = mdss_get_icn6211_i2c_client();
						if (!ctrl_pdata->i2c_handle)
						{
							pr_err("ctrl_pdata->i2c_handle is null\n");
								//goto error;   /*modified by fangchengbing  return error will lead to crash*/
						}
					
						mdss_i2c_parse_dcs_cmds(np, &ctrl_pdata->i2c_on_cmds, "qcom,mdss-i2c-on-command");
						
				} else {  /*normal flow*/
					//....
				}
		
		}


		/**
		 * @brief parse i2c command in dtsi file, format [%d %d] ==> [reg, val]
		 * store them in struct i2c_cmd_list
		 *
		 * @param np
		 * @param pcmds where i2c will store
		 * @param cmd_key flag contain i2c command
		 *
		 * @return 
		 */
		static int mdss_i2c_parse_dcs_cmds(struct device_node *np,
				struct i2c_cmd_list *pcmds, char *cmd_key)
		{
			const char *data;
			int blen = 0;
			char *buf;
			int i = 0, j = 0;
		
			data = of_get_property(np, cmd_key, &blen);
			if (!data) {
				pr_err("%s: failed, key=%s\n", __func__, cmd_key);
				return -ENOMEM;
			}
		
			if (blen < 0 && blen / 2 == 1)
				pr_err("%s format err\n", cmd_key);
		
			pr_debug("mdss_i2c_parse_dcs_cmds blen = %d\n", blen);
			buf = kzalloc(sizeof(char) * blen, GFP_KERNEL);
			if (!buf)
				return -ENOMEM;
		
			memcpy(buf, data, blen);
			pcmds->cmds = kzalloc(sizeof(struct i2c_ctrl_hdr) * blen / 2, GFP_KERNEL);
			pcmds->cmds_cnt = blen / 2;
			for (i = 0; i < blen; i+=2)
			{
				j = i / 2;
				pcmds->cmds[j].reg = buf[i];
				pcmds->cmds[j].val = buf[i+1];
			}
		
			return 0;
		}


2. 控制器初始化时，注册屏初始化命令表:

		void mdss_dsi_ctrl_init(struct mdss_dsi_ctrl_pdata *ctrl)
		{
			//...
			if (ctrl->cmd_access == CMD_ACCESS_DSI)
				ctrl->cmdlist_commit = mdss_dsi_cmdlist_commit;
			else if (ctrl->cmd_access == CMD_ACCESS_I2C)
				ctrl->cmdlist_commit = mdss_i2c_cmdlist_commit;
			//...
		}

		/**
		 * @brief  write i2c command to mipi@rgb chip
		 *
		 * @param ctrl include i2c_client
		 * @param from_mdp not important
		 *
		 * @return 
		 */
		int mdss_i2c_cmdlist_commit(struct mdss_dsi_ctrl_pdata *ctrl, int from_mdp)
		{
			int rc = 0;
			int i = 0;
			struct i2c_cmd_list on_cmds;
			on_cmds = ctrl->i2c_on_cmds;
			mutex_lock(&ctrl->cmd_mutex);
			if (ctrl->i2c_handle) {
				for (i = 0; i < on_cmds.cmds_cnt; i++)
				{
					rc = i2c_smbus_write_byte_data(ctrl->i2c_handle, on_cmds.cmds[i].reg, on_cmds.cmds[i].val);
					if (rc < 0) {
						mutex_unlock(&ctrl->cmd_mutex);
						return rc;
					}
				}
			}
		
			mutex_unlock(&ctrl->cmd_mutex);
			return rc;
		}


3. 下发屏参数命令

		static int mdss_dsi_panel_on(struct mdss_panel_data *pdata)
		{
			//...
			if (ctrl->cmd_access == CMD_ACCESS_DSI) {
				if (ctrl->on_cmds.cmd_cnt)
					mdss_dsi_panel_cmds_send(ctrl, &ctrl->on_cmds);
			} else if (ctrl->cmd_access == CMD_ACCESS_I2C) {
				if (ctrl->i2c_on_cmds.cmds_cnt)
					ctrl->cmdlist_commit(ctrl, 0);
			}
			//...
		}


***注意!***

最重要的有一点不要忘记，平台需要利用 I2C 总线去下发屏初始化参数，根据 I2C 协议，需要给定 I2C 设备在总线上的地址：

在 bootloader 里直接配置上 0x2C 即可，在 kernel 里面，dtsi 中配置如下:

	i2c@78b6000 {
	      icn6211_mipi_rgb@2C {
			        compatible = "qcom,icn6211_mipi_rgb";
					      reg = <0x2C>;
						      };
	
		    };


## 总结

其实 MIPI 转 RGB 的修改点不多，也不难。就一个地方：初始化命令通过 MIPI 的 DATA0 下发，转移到了 I2C 下发。
