---
layout:     post
title:      LINUX下的LCD驱动层次分析
subtitle:   LINUX-LCD
date:       2018-7-14
author:     Muggle
header-img:
catalog: 	 true
tags:
    - Linux驱动
---

**浅析**[http://www.cnblogs.com/lifexy/p/7603327.html](http://www.cnblogs.com/lifexy/p/7603327.html)

**详细**[http://www.cnblogs.com/lifexy/p/7604011.html](http://www.cnblogs.com/lifexy/p/7604011.html)
# 大体步骤：
## 在驱动init入口函数中:

	1)分配一个fb_info结构体
	
	2)设置fb_info
	
	　　2.1)设置固定的参数fb_info-> fix
	
	　　2.2) 设置可变的参数fb_info-> var
	
	　　2.3) 设置操作函数fb_info-> fbops
	
	　　2.4) 设置fb_info 其它的成员
	
	3)设置硬件相关的操作    
	
	　　3.1)配置LCD引脚
	
	　　3.2)根据LCD手册设置LCD控制器
	
	　　3.3)分配显存(framebuffer),把地址告诉LCD控制器和fb_info
	
	4)开启LCD,并注册fb_info: register_framebuffer()
	
	　　4.1) 直接在init函数中开启LCD(后面讲到电源管理,再来优化)
	
	　　　　控制LCDCON5允许PWREN信号,
	
	　　　　然后控制LCDCON1输出PWREN信号,
	
	　　　　输出GPB0高电平来开背光,
	
	　　4.2) 注册fb_info

 

## 在驱动exit出口函数中:

	1)卸载内核中的fb_info
	
	2) 控制LCDCON1关闭PWREN信号,关背光,iounmap注销地址
	
	3)释放DMA缓存地址dma_free_writecombine()
	
	4)释放注册的fb_info
	/* LCD control */ 

## 代码：
	 struct  lcd_reg{  
	              unsigned long    lcdcon1; 
	              unsigned long       lcdcon2;  
	              unsigned long       lcdcon3; 
	              unsigned long       lcdcon4; 
	              unsigned long       lcdcon5; 
	              unsigned long       lcdsaddr1;     
	              unsigned long       lcdsaddr2;     
	              unsigned long       lcdsaddr3 ;
	              unsigned long       redlut;   
	              unsigned long       greenlut;
	              unsigned long       bluelut;
	              unsigned long       reserved[9];  
	              unsigned long       dithmode;     
	              unsigned long       tpal ;            
	              unsigned long       lcdintpnd;
	              unsigned long       lcdsrcpnd;
	              unsigned long       lcdintmsk;
	              unsigned long       tconsel;  
	};
	
	static struct lcd_reg  *lcd_reg;
	 
	static struct fb_info *my_lcd;    //定义一个全局变量
	static u32 pseudo_palette[16];   //调色板数组,被fb_info->pseudo_palette调用
	
	static inline unsigned int chan_to_field(unsigned int chan, struct fb_bitfield *bf)
	{
	/*内核中的单色都是16位,默认从左到右排列,比如G颜色[0x1f],那么chan就等于0XF800*/
	       chan       &= 0xffff;
	       chan       >>= 16 - bf->length;    //右移,将数据靠到位0上
	       return chan << bf->offset;    //左移一定偏移值,放入16色数据中对应的位置
	}
	
	static int my_lcdfb_setcolreg(unsigned int regno, unsigned int red,unsigned int green, unsigned int blue,unsigned int transp, struct fb_info *info)      //设置调色板函数,供内核调用
	{
	       unsigned int val;      
	       if (regno >=16)                //调色板数组不能大于15
	              return 1;
	
	       /* 用red,green,blue三个颜色值构造出16色数据val */
	       val  = chan_to_field(red,      &info->var.red);
	       val |= chan_to_field(green, &info->var.green);
	       val |= chan_to_field(blue,      &info->var.blue);
	      
	       ((u32 *)(info->pseudo_palette))[regno] = val;     //放到调色板数组中
	       return 0;
	}
	
	
	static struct fb_ops my_lcdfb_ops = {
	      .owner           = THIS_MODULE,
	      .fb_setcolreg  = my_lcdfb_setcolreg,//调用my_lcdfb_setcolreg()函数,来设置调色板fb_info-> pseudo_palette
	      .fb_fillrect       = cfb_fillrect,     //填充矩形
	      .fb_copyarea   = cfb_copyarea,     //复制数据
	      .fb_imageblit  = cfb_imageblit,    //绘画图形,
	};
	
	static int lcd_init(void)
	{
	  /*1.申请一个fb_info结构体*/
	  my_lcd= framebuffer_alloc(0,0); 
	
	  /*2.设置fb_info*/
	 
	  /* 2.1设置固定的参数fb_info-> fix */
	  /*my_lcd->fix.smem_start    物理地址后面注册MDA缓存区设置*/ 
	  strcpy(my_lcd->fix.id, "mylcd");                       //名字
	  my_lcd->fix.smem_len =LCD_xres*LCD_yres*2;             //地址长
	  my_lcd->fix.type        =FB_TYPE_PACKED_PIXELS;
	  my_lcd->fix.visual            =FB_VISUAL_TRUECOLOR;            //真彩色
	  my_lcd->fix.line_length      =LCD_xres*2;               //LCD 一行的字节
	
	  /* 2.2 设置可变的参数fb_info-> var  */
	  my_lcd->var.xres        =LCD_xres;                          //可见屏X 分辨率
	  my_lcd->var.yres        =LCD_yres;                          //可见屏y 分辨率
	  my_lcd->var.xres_virtual     =LCD_xres;                   //虚拟屏x分辨率
	  my_lcd->var.yres_virtual     =LCD_yres;                   //虚拟屏y分辨率
	  my_lcd->var.xoffset           = 0;                     //虚拟到可见屏幕之间的行偏移
	  my_lcd->var.yoffset           =0;                      //虚拟到可见屏幕之间的行偏移
	
	  my_lcd->var.bits_per_pixel=16;                //像素为16BPP
	  my_lcd->var.grayscale       =   0;            //灰色比例
	 
	  my_lcd->var.red.offset      =     11;
	  my_lcd->var.red.length      =     5;
	  my_lcd->var.green.offset  =       5;
	  my_lcd->var.green.length  =       6;
	  my_lcd->var.blue.offset     =     0;
	  my_lcd->var.blue.length     =     5;
	
	/* 2.3 设置操作函数fb_info-> fbops  */
	  my_lcd->fbops                 = &my_lcdfb_ops;
	
	
	  /* 2.4 设置fb_info 其它的成员  */ 
	 /*my_lcd->screen_base    虚拟地址在后面注册MDA缓存区设置*/
	  my_lcd->pseudo_palette   =pseudo_palette;            //保存调色板数组
	  my_lcd->screen_size          =LCD_xres * LCD_yres *2;     //虚拟地址长
	
	  /*3   设置硬件相关的操作*/
	  /*3.1 配置LCD引脚*/
	  GPBcon                     = ioremap(0x56000010, 8);
	  GPBdat                     = GPBcon+1;
	  GPCcon                     = ioremap(0x56000020, 4);
	  GPDcon              　　　　= ioremap(0x56000030, 4);
	  GPGcon            　　　　  = ioremap(0x56000060, 4);
	
	  *GPBcon            &=~(0x03<<(0*2));
	  *GPBcon            |= (0x01<<(0*2));     //PGB0背光
	  *GPBdat            &=~(0X1<<0);           //关背光
	  *GPCcon            =0xaaaaaaaa;
	  *GPDcon            =0xaaaaaaaa;
	  *GPGcon            |=(0x03<<(4*2));    //GPG4:LCD信号
	
	  /*3.2 根据LCD手册设置LCD控制器,参考之前的裸机驱动*/
	  lcd_reg=ioremap(0X4D000000, sizeof( lcd_reg) );
	   /*HCLK:100Mhz */
	   lcd_reg->lcdcon1     = (4<<8) | (0X3<<5) |  (0x0C<<1) ;
	   lcd_reg->lcdcon2     = ((3)<<24) | (271<<14) |  ((1)<<6) |((0)<<0);
	   lcd_reg->lcdcon3     = ((16)<<19) | (479<<8) | ((10));
	   lcd_reg->lcdcon4     = (4);
	   lcd_reg->lcdcon5     = (1<<11) | (1<<9) | (1<<8) |(1<<0);     
	
	   lcd_reg->lcdcon1     &=~(1<<0);              // 关闭PWREN信号输出
	   lcd_reg->lcdcon5     &=~(1<<3);              //禁止PWREN信号
	
	 /* 3.3  分配显存(framebuffer),把地址告诉LCD控制器和fb_info*/
	   my_lcd->screen_base=dma_alloc_writecombine(0,my_lcd->fix.smem_len,  &my_lcd->fix.smem_start, GFP_KERNEL);
	
	   /*lcd控制器的地址必须是物理地址*/
	   lcd_reg->lcdsaddr1 =(my_lcd->fix.smem_start>>1)&0X3FFFFFFF;  //保存缓冲起始地址A[30:1]  
	   lcd_reg->lcdsaddr2 =((my_lcd->fix.smem_start+my_lcd->screen_size)>>1)&0X1FFFFF; //保存存缓冲结束地址A[21:1]
	   lcd_reg->lcdsaddr3 =LCD_xres& 0x3ff;　　　　　　　　//OFFSIZE[21:11]:保存LCD上一行结尾和下一行开头的地址之间的差
	　　　　　　　　　　　　　　　　　　　　　　　　　　　　　　  //PAGEWIDTH [10:0]:保存LCD一行占的宽度(半字数为单位)
	
	   /*4开启LCD,并注册fb_info: register_framebuffer()*/
	   /*4.1 直接在init函数中开启LCD(后面讲到电源管理,再来优化)*/
	  lcd_reg->lcdcon1      |=1<<0;         //输出PWREN信号
	  lcd_reg->lcdcon5      |=1<<3;        //允许PWREN信号
	  *GPBdat                   |=(0X1<<0);           //开背光
	
	   /*4.2 注册fb_info*/
	   register_framebuffer(my_lcd);
	   return 0;
	}
	
	static int lcd_exit(void)
	{
	   /* 1卸载内核中的fb_info*/
	     unregister_framebuffer(my_lcd);
	
	  /*2 控制LCDCON1关闭PWREN信号,关背光,iounmap注销地址*/
	   lcd_reg->lcdcon1 &=~(1<<0);              // 关闭PWREN信号输出
	   lcd_reg->lcdcon5 &=~(1<<3);            //禁止PWREN信号
	   *GPBdat            &=~(0X1<<4);           //关背光
	   iounmap(GPBcon);
	   iounmap(GPCcon);
	   iounmap(GPDcon);
	   iounmap(GPGcon);
	
	  /*3.释放DMA缓存地址dma_free_writecombine()*/
	  dma_free_writecombine(0,my_lcd->screen_size,my_lcd->screen_base,my_lcd->fix.smem_start);
	
	   /*4.释放注册的fb_info*/
	   framebuffer_release(my_lcd);
	
	   return 0;
	}
	
	module_init(lcd_init);
	module_exit(lcd_exit);
	MODULE_LICENSE("GPL");