# Framebuffer程序

```c
#include <sys/mman.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <linux/fb.h>
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/ioctl.h>
#include "font_8x16.h"

#define COLOR_WHITE 0xFFFFFF
#define COLOR_BLACK 0x000000
#define COLOR_RED 0xFF0000
#define COLOR_GREEN 0x00FF00
#define COLOR_BLUE 0x0000FF

// 定义默认背景色
#define BACKGROUND_COLOR COLOR_BLACK

int fd_fb;
struct fb_var_screeninfo var; /* Current var */
int screen_size;
unsigned char *fbmem;
unsigned int line_width;
unsigned int pixel_width;

int fd_hzk16;

/**********************************************************************
 * 函数名称： lcd_put_pixel
 * 功能描述： 在LCD指定位置上输出指定颜色（描点）
 * 输入参数： x坐标，y坐标，颜色
 * 输出参数： 无
 * 返 回 值： 会
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2020/05/12	     V1.0	  zh(angenao)	      创建
 ***********************************************************************/
void lcd_put_pixel(int x, int y, unsigned int color)
{
	unsigned char *pen_8 = fbmem + y * line_width + x * pixel_width;
	unsigned short *pen_16;
	unsigned int *pen_32;

	unsigned int red, green, blue;

	pen_16 = (unsigned short *)pen_8;
	pen_32 = (unsigned int *)pen_8;

	// 根据不同设备处理对应的像素点的颜色值
	switch (var.bits_per_pixel)
	{
	case 8:
	{
		*pen_8 = color;
		break;
	}
	case 16:
	{
		/* 565 */
		red = (color >> 16) & 0xff;
		green = (color >> 8) & 0xff;
		blue = (color >> 0) & 0xff;
		color = ((red >> 3) << 11) | ((green >> 2) << 5) | (blue >> 3);
		*pen_16 = color;
		break;
	}
	case 32:
	{
		*pen_32 = color;
		break;
	}
	default:
	{
		printf("can't surport %dbpp\n", var.bits_per_pixel);
		break;
	}
	}
}

/**********************************************************************
 * 函数名称： draw_solid_rectangle
 * 功能描述： 在LCD指定位置上输出一个实心矩形
 * 输入参数： x坐标，y坐标，矩形宽度、矩形高度、颜色
 * 输出参数： 无
 * 返 回 值： 会
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2023/10/26	     V1.0	  Treeleaf	      创建
 ***********************************************************************/
void draw_solid_rectangle(int x, int y, int width, int height, unsigned int color)
{
	int i, j;

	for (i = x; i < x + width; i++)
	{
		for (j = y; j < y + height; j++)
		{
			lcd_put_pixel(i, j, color);
		}
	}
}

/**********************************************************************
 * 函数名称： draw_hollow_rectangle
 * 功能描述： 在LCD指定位置上输出一个空心矩形
 * 输入参数： x坐标，y坐标，矩形宽度、矩形高度、颜色、框架宽度
 * 输出参数： 无
 * 返 回 值： 会
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2023/10/26	     V1.0	  Treeleaf	      创建
 ***********************************************************************/
void draw_hollow_rectangle(int x, int y, int width, int height, unsigned int color, int line_width)
{
	for (int i = x; i < x + width; i++)
	{
		for (int j = 0; j < line_width; j++)
		{
			lcd_put_pixel(i, y + j, color);			 // 顶边
			lcd_put_pixel(i, y + height - j, color); // 底边
		}
	}

	for (int i = y; i < y + height; i++)
	{
		for (int j = 0; j < line_width; j++)
		{
			lcd_put_pixel(x + j, i, color);			// 左边
			lcd_put_pixel(x + width - j, i, color); // 右边
		}
	}
}

/**********************************************************************
 * 函数名称： draw_filled_circle
 * 功能描述： 在LCD指定位置上输出一个实心圆
 * 输入参数： 圆心x坐标，圆心y坐标、圆的半径，颜色
 * 输出参数： 无
 * 返 回 值： 会
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2023/10/26	     V1.0	  Treeleaf	      创建
 ***********************************************************************/
void draw_filled_circle(int x0, int y0, int radius, unsigned int color)
{
	int x = radius;
	int y = 0;
	int radius_error = 1 - x;

	while (x >= y)
	{
		for (int i = x0 - x; i <= x0 + x; i++)
		{
			for (int j = y0 - y; j <= y0 + y; j++)
			{
				lcd_put_pixel(i, j, color);
			}
		}

		for (int i = x0 - y; i <= x0 + y; i++)
		{
			for (int j = y0 - x; j <= y0 + x; j++)
			{
				lcd_put_pixel(i, j, color);
			}
		}

		y++;

		if (radius_error < 0)
		{
			radius_error += 2 * y + 1;
		}
		else
		{
			x--;
			radius_error += 2 * (y - x + 1);
		}
	}
}

/**********************************************************************
 * 函数名称： lcd_put_ascii
 * 功能描述： 在LCD指定位置上显示一个8*16的字符
 * 输入参数： x坐标，y坐标，ascii码，字符颜色
 * 输出参数： 无
 * 返 回 值： 无
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2020/05/12	     V1.0	  zh(angenao)	      创建
 ***********************************************************************/
void lcd_put_ascii(int x, int y, unsigned char c, int color)
{
	unsigned char *dots = (unsigned char *)&fontdata_8x16[c * 16];
	int i, b;
	unsigned char byte;

	for (i = 0; i < 16; i++)
	{
		byte = dots[i];
		for (b = 7; b >= 0; b--)
		{
			// 从最高位开始解析
			if (byte & (1 << b))
			{
				/* show */
				lcd_put_pixel(x + 7 - b, y + i, color); /* 白 */
			}
			else
			{
				/* hide */
				lcd_put_pixel(x + 7 - b, y + i, BACKGROUND_COLOR); /* 黑 */
			}
		}
	}
}

/**********************************************************************
 * 函数名称： lcd_put_string
 * 功能描述： 在LCD指定位置上显示一个8*16的字符串
 * 输入参数： x坐标，y坐标，字符串，字符串颜色
 * 输出参数： 无
 * 返 回 值： 无
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2020/05/12	     V1.0	  zh(angenao)	      创建
 ***********************************************************************/

void lcd_put_string(int x, int y, char *str, int color)
{
	int len = strlen(str); // 获取字符串的长度

	for (int i = 0; i < len; i++)
	{
		char character = str[i]; // 通过下标索引取出字符
		lcd_put_ascii(x + 8 * i, y, character, color);
	}
}

/**********************************************************************
 * 函数名称： lcd_clear
 * 功能描述： 清屏
 * 输入参数： 颜色
 * 输出参数： 无
 * 返 回 值： 无
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2020/05/12	     V1.0	  zh(angenao)	      创建
 ***********************************************************************/
void lcd_clear(int color)
{
	memset(fbmem, color, screen_size);
}

/**********************************************************************
 * 函数名称： lcd_put_chinese
 * 功能描述： 在LCD指定位置上显示一个16*16的汉字
 * 输入参数： x坐标，y坐标，ascii码,HZK16的文件描述符
 * 输出参数： 无
 * 返 回 值： 无
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2020/05/12	     V1.0	  zh(angenao)	      创建
 ***********************************************************************/
// TODO 由全局文件描述符来取汉字点阵数据
void lcd_put_chinese(int x, int y, unsigned char *str, int HZK16_fd, int color)
{
	unsigned int area = str[0] - 0xA1;
	unsigned int where = str[1] - 0xA1;
	unsigned char buf[32];
	unsigned char byte;
	lseek(HZK16_fd, (area * 94 + where) * 32, SEEK_SET);
	read(HZK16_fd, buf, 32);

	int i, j, b;
	for (i = 0; i < 16; i++)
		for (j = 0; j < 2; j++)
		{
			byte = buf[i * 2 + j];
			for (b = 7; b >= 0; b--)
			{
				if (byte & (1 << b))
				{
					/* show */
					lcd_put_pixel(x + j * 8 + 7 - b, y + i, color); /* 字体颜色 */
				}
				else
				{
					/* hide */
					lcd_put_pixel(x + j * 8 + 7 - b, y + i, BACKGROUND_COLOR); /* 背景色 */
				}
			}
		}
}

/**********************************************************************
 * 函数名称： lcd_put_chinese_string
 * 功能描述： 在LCD指定位置上显示一个16*16的汉字字符串
 * 输入参数： x坐标，y坐标，ascii码,HZK16的文件描述符
 * 输出参数： 无
 * 返 回 值： 会
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2023/10/26	     V1.0	  Treeleaf	      创建
 ***********************************************************************/
void lcd_put_chinese_string(int x, int y, unsigned char *str, int HZK16_fd, int color)
{
	int len = strlen(str);
	printf("%d\n", len);
	int i;
	for (i = 0; i < len / 2; i++)
	{
		char buf[2] = {str[i * 2], str[i * 2 + 1]};
		lcd_put_chinese(x, y, buf, HZK16_fd, color);
		x += 16;
	}
}
/**********************************************************************
 * 函数名称： lcd_put_mixed_string
 * 功能描述： 在LCD指定位置上显示一个中文英文混合字符串，支持换行显示'\n'
 * 输入参数： x坐标，y坐标，ascii码,HZK16的文件描述符,字符串颜色
 * 输出参数： 无
 * 返 回 值： 会
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2023/10/26	     V1.0	  Treeleaf	      创建
 ***********************************************************************/
void lcd_put_mixed_string(int x, int y, char *str, int HZK16_fd, int color)
{
	int len = strlen(str);
	int i;
	int x_tmp = x;
	for (i = 0; i < len; i++)
	{
		unsigned char character = str[i];
		if (character >= 0x80)
		{
			// 如果字符大于等于0x80，表示它是汉字（或其他非ASCII字符）
			// 调用 lcd_put_chinese 函数来显示汉字
			unsigned char chinese_char[2] = {character, str[i + 1]};
			lcd_put_chinese(x, y, chinese_char, HZK16_fd, color);
			i++;	 // 跳过下一个字符，因为它已经被处理了
			x += 16; // 假设汉字字符占用16个像素的宽度
		}
		else if (character == '\n')
		{
			// 如果字符是换行符，将 y 坐标增加16，并重置 x 坐标为初始坐标
			y += 16;
			x = x_tmp;
		}
		else
		{
			// 如果字符是ASCII字符，调用 lcd_put_ascii 函数来显示
			lcd_put_ascii(x, y, character, color);
			x += 8; // ASCII字符通常占用8个像素的宽度
		}
	}
}

/**********************************************************************
 * 函数名称： lcd_init
 * 功能描述： LCD初始化函数
 * 输入参数： 无
 * 输出参数： 无
 * 返 回 值： 错误码
 * 修改日期        版本号     修改人	      修改内容
 * -----------------------------------------------
 * 2023/10/26	     V1.0	  Treeleaf	      创建
 ***********************************************************************/
int lcd_init(void)
{
	fd_fb = open("/dev/fb0", O_RDWR);
	if (fd_fb < 0)
	{
		printf("can't open /dev/fb0\n");
		return -1;
	}

	if (ioctl(fd_fb, FBIOGET_VSCREENINFO, &var))
	{
		printf("can't get var\n");
		return -1;
	}

	line_width = var.xres * var.bits_per_pixel / 8;
	pixel_width = var.bits_per_pixel / 8;
	screen_size = var.xres * var.yres * var.bits_per_pixel / 8;
	fbmem = (unsigned char *)mmap(NULL, screen_size, PROT_READ | PROT_WRITE, MAP_SHARED, fd_fb, 0);
	if (fbmem == (unsigned char *)-1)
	{
		printf("can't mmap\n");
		return -1;
	}

	fd_hzk16 = open("HZK16", O_RDONLY);
	if (fd_hzk16 < 0)
	{
		printf("can't open HZK16\n");
		return -1;
	}
	return 0;
}

int main(int argc, char **argv)
{
	if (lcd_init() != 0)
	{
		return -1;
	}

	/* 清屏: 全部设为黑色 */
	lcd_clear(COLOR_BLACK);

	lcd_put_mixed_string(400, 400 + 48, "来试试换行\n然后这是第二行\nThis is line3", fd_hzk16, COLOR_RED);
	munmap(fbmem, screen_size);
	close(fd_fb);
	close(fd_hzk16);
	return 0;
}
```

​![IMG_20231026_161043](assets/IMG_20231026_161043-20231026161116-5orpuht.jpg)​

‍
