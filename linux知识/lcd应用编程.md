framebuffer：用来保存一帧数据的内存

framebuffer中会保存lcd屏幕上每一个像素点的值，他里面的每一块数据和像素点都是一一对应的。

应用程序值需要将数据写进framebuffer中即可，由lcd控制器将buffer中的数据在屏幕上显示出来，驱动程序设置好lcd控制器后，他就会自动的把buffer中像素的值拿出来，使它在屏幕上显示出来。去数据时从左往右，从上到下，取完一帧后又回到开始的地方，周而复始。

#### lcd应用程序框架

应用程序修改lcd某一个点的颜色使，秩序要知道：

1.bpp每一个像素点用多少位来表示，格式使怎么样的

2，分辨率：要计算像素点的位置和buffer的对应数据的位置

公式：（y*xres）*bpp/8+x * yres * bpp/8得到像素对应的内存在buffer中的偏移地址，在结合基地址就可以了

应用程序编写流程：

![](E:\课程及内核源码\01_all_series_quickstart\09_u-boot完全分析与移植\doc_pic\pic\2025-05-08-20-46-28-image.png)

linux内核中，要提供lcd的两类参数，一类是可变参数，一类是固定参数

应用程序主要关心可变参数：

1，分辨率

2，bpp

3，bpp的格式，也就是位域，也就是rgb的比例，这是在可变参数中设置的

![](E:\课程及内核源码\01_all_series_quickstart\09_u-boot完全分析与移植\doc_pic\pic\2025-05-08-20-50-10-image.png)

项目中应用程序编写：

设计函数

1，open：打开lcd代表文件，得到文件描述符，以便后续操作

2，ioctl：获取lcd可变参数

可以使用以下代码获取 fb_var_screeninfo：

```
12 static struct fb_var_screeninfo var; /* Current var */
……

79 if (ioctl(fd_fb, FBIOGET_VSCREENINFO, &var))

80 {

81 printf("can't get var\n");

82 return -1;

83 }
```

注意到 ioctl 里用的参数是：FBIOGET_VSCREENINFO，它表示 get var screen

info，获得屏幕的可变信息；当然也可以使用 FBIOPUT_VSCREENINFO 来调整这

些参数，但是很少用到。

3,mmap:映射内存,映射后直接操作fb_base即可

```
line_width = var.xres * var.bits_per_pixel / 8;
86 pixel_width = var.bits_per_pixel / 8;
87 screen_size = var.xres * var.yres * var.bits_per_pixel / 8;
88 fb_base = (unsigned char *)mmap(NULL , screen_size, PROT_READ | PROT_WRITE, M
AP_SHARED, fd_fb, 0);
89 if (fb_base == (unsigned char *)-1)
90 {
91 printf("can't mmap\n");
92 return -1;
93 }
```

screen_size 是整个 Framebuffer 的大小；PROT_READ |

PROT_WRITE 表示该区域可读、可写；MAP_SHARED 表示该区域是共享的，APP 写

入数据时，会直达驱动程序,

4，描点函数：能够描点，就能写字，就能划线

原理：显存的基地址加上偏移值就找到了对应的内存，直接写入数据即可。

写入数据时，要完成颜色格式的转换，以为color是32位的数据，bpp一般不是32位的

偏移值：（y*xres）*bpp/8+x * yres * bpp/8

#### 升级：由描点到写字符和文字

1，显示什么字符：要获取字符的编码值，在不同的编码方式（ASCII，ANSI等）中，这个值是不同的。

2，找到字体文件，要将字符显示成什么样子还需要考虑使用什么字体。

##### ASCII码的点阵字符的显示

1，得到编码：char c=‘A’;

2,得到点阵：linux内核中提供了点阵，比如linux-4.9.88\lib\fonts \font_8x16.c文件

里面的fontdata数组中存放的就是字符点阵，这个文件中每个字符占据16个字节，每个字节分别对应像素中的16行，一个字节对应一行里面的八个像素

![](E:\课程及内核源码\01_all_series_quickstart\09_u-boot完全分析与移植\doc_pic\pic\2025-05-09-15-06-18-image.png)

3，程序：

3.1 根据编码值，在数组中找到字符对应的点阵，也就是16个字节的数据，然后循环处理每个字节

3.2 处理每一个字节：取出字节中的每一位像素，根据那一位的值点亮和熄灭像素

4，显示单个字符，可以选择颜色

5，显示字符串，可以换行

##### 显示中文字符

1，显示中文字符时，**必须注意字符的编码格式**。

-finput-charset=GB2312 -finput-charset=UTF-8 在编译命令中加上上面两条语句，转化编码格式

2，查找汉字字库，得到汉字的点阵表示：**使用编码值找到在数组中的位置**

从网上搜到 HZK16 这个文件，它是**常用汉字**的 16*16 点阵字库。HZK16里每个汉字使用 32 字节来描述，每个字节中每一位用来表示一个像素，位值等于 1 时表示对应像素被点亮，位值等于 0 时表示对应像素被熄灭。

HZK16 中是以 GB2312 编码值来查找点阵的，以“中”字为例，它的编码值

是“**0xd6 0xd0**”，其中的 0xd6 表示“**区码**”，表示在哪一个区：第“0xd6 - 0xa1”

区；其中的 0xd0 表示“**位码**”，表示它是这个区里的哪一个字符：第“**0xd0 -**

**0xa1**”个。每一个区有 94 个汉字。区位码从 0xa1 而不是从 0 开始，是为了兼

容 ASCII 码。所以，我们要显示的“**中”字**，它的 GB2312 编码是 d6d0，它是 HZK16 里

第“**(0xd6-0xa1)*94+(0xd0-0xa1)**”个字符。

3，程序：循环将点阵中的数据转化为像素

3.1 打开汉字库文件 open

3.2 获得文件大小 fstat以便后续mmap使用

##### 交叉编译库

程序运行时依赖各种库，嵌入式Linux也是，但需要将库编译到开发板上

freetype是一个矢量字符库

手工编译安装，适用于小程序，以及依赖的库比较少时更方便

##### 使用freetype显示单个文字

点阵字体大小固定，不易缩放，缩放后会产生锯齿

矢量字体：易于缩放

1. 确定关键点，

2. 使用数学曲线（贝塞尔曲线）连接头键点，

3. 填充闭合区线内部空间。

freetype帮助实现上面内容

写程序时只需：给定编码，设置参数
