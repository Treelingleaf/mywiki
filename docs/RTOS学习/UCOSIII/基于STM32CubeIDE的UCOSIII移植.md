# 基于STM32CubeIDE的UCOSIII移植

参考：[【经验分享】STM32CubeIDE使用说明—— 运行µC/OS-III](https://shequ.stmicroelectronics.cn/thread-634895-1-1.html)

# 1. **下载UCOSIII源码并解压**​

​可以选择去官网下载，但是相对麻烦，因为需要注册，已知国内的.com邮箱不能注册，所以还是推荐从别人分享的连接下载，这里给出连接：

> 连接：[百度网盘 请输入提取码](https://pan.baidu.com/s/1L3eoviykjQmNmDQqmPSmiA?pwd=d70z)  提取码：d70z

这里还是以`STM32F103C8T6`​处理器来作为说明，官网提供的例程里面只包含的部分芯片，但是不重要，只要找到内核相同的都可以使用。例如STM32F103C8T6 属于Arm `Cortex-M3`​，那我们只需要下载例程里面相同内核的芯片STM32F107的例程就可以使用，将其解压后即可得到源文件。

​![image](assets/image-20231010224634-afn8ubr.png)

---

 

# 2. **用STM32CubeIDE新建一个工程并添加UCOS源码**

如何新建工程这里就不赘述了，网上教程多得是。

新建的默认工程目录：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694875533406-98adc3c1-0e49-4740-9146-c5138e2e49da.png)​

这里可以在工程根目录（也就是**上图位置**）新建一个`RTOS`​文件夹，用来存放UCOSIII的源码。

‍

然后在`RTOS`​目录里新建一个`uC-Config`​文件夹用来存放UCOSIII的板级配置文件：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694875790357-8696403a-f67a-4028-a599-41089b172b31.png)​

然后将下图目录的**八个选中的文件**放入刚才新建的uC-Config目录内：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694875958327-23854fda-93bd-4386-988d-366a5ae94f17.png)​

同时把下图目录的`bsp.c`​和`bsp.h`​也拷贝到`uC-Config`​目录：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694876056516-910117ab-8848-47fa-9070-5c13a40d183a.png)​

拷贝工作到此结束，UCOSIII相关的文件已经全部拷到工程文件夹里了，接下来就是具体配置了。

---

 

# 3. **工程具体配置**

进入STM32CubeIDE，刷新工程，就能看见刚才配置的文件了：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694876341458-9d298274-8b48-4557-8ab3-a8d595884319.png)​

**首先需要配置环境变量也就是头文件目录：**

**找到如下设置界面：** 对工程右键-->菜单最下面的属性

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694876488121-4b2924ea-0ae2-4b05-9c32-eb2ef3212fd7.png)​

在上面的`Include paths`​添加刚才UCOSIII的头文件目录：

首先是`uC-Config`​目录

![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694876674562-9571c79c-408b-4228-ac97-eb6abd8e32f0.png)​

* 然后是`\RTOS\uC-CPU\ARM-Cortex-M3`​目录下的`GNU`​文件夹，至于为什么选择GNU，因为不同得到IDE的编译器不一样，产生的汇编文件不一样，可以理解成选择STM32CubeIDE的编译器GUN,提一嘴MDK的话，这里需要选择`RealView`​。

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694876752860-c361eead-9d46-4bfa-8be7-388726e59469.png)​

其他的就不一一演示了，反正就是查看那几个文件夹哪有.h文件都添加进来，遇见上面上个**分支的都选GNU**，看最终结果：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694877116040-6a0f5993-aaaa-492a-a0c3-4e53471b3285.png)​

还需要在下面位置也加入头文件路径，和上面是一样的：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694879048128-195f52a9-7f81-4ff5-b219-de1d2bc84c61.png)​

在这加入RTOS文件路径：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694879088432-334860ef-ce1f-44c0-bf02-a51ad4c56072.png)​

​接下来对工程内的RTOS文件夹进行操作：

​`右键->Resource Configurations->Exclude from Build...`​

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694877285700-2fc19f4f-9aca-4452-be91-39040f29c698.png)​

取消勾选Debug，作用就是把该目录加入Debug的编译。

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694877364884-cc0d0d7c-4f1a-4e56-b1ab-57cf1eddf25a.png)​

利用上面的方法把一些不需要的文件夹取消编译，就是刚才添加头文件路径的时候，三选一没选中的目录，也是不需要编译的，当然也可以直接删除：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694877552320-f30ce77f-519c-4a40-b7e3-2bc4dfd5cbf0.png)​

至此，第三步完成了。

---

 

# 4. **修改源码**

如果尝试编译，会报大量错误，因为`bsp.c`​和`bsp.h`​文件中有太多不相关的代码。首先，将bsp.h文件修改成下面内容：

```c

#ifndef  BSP_PRESENT
#define  BSP_PRESENT


/*
*********************************************************************************************************
*                                                 EXTERNS
*********************************************************************************************************
*/

#ifdef   BSP_MODULE
#define  BSP_EXT
#else
#define  BSP_EXT  extern
#endif

#include <cpu.h>
#include <cpu_core.h>
#include "stm32f1xx_hal.h"
#endif
```

然后 ，`bsp.c`​文件夹中有很多例程平台中的LED配置和操作的代码，全部删除。保留如下：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694878094247-7156fb9a-3d0f-4c76-a1b1-02ece935ef95.png)​

重写`BSP_CPU_ClkFreq`​函数，获取CPU主频：

```c
CPU_INT32U  BSP_CPU_ClkFreq (void)
{
	CPU_INT32U hclk_freq;
	hclk_freq=HAL_RCC_GetHCLKFreq();//HAL库的API
	return hclk_freq;
}
```

打开`uC-Config`​文件夹下的`lib_cfg.h`​文件，找到下图所示的代码， **并将27调整为5**，减少UCOS内存方向程序占用的RAM空间。

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694878482001-1dfd3fe3-6284-4ffe-b8bd-c6c3c6fa00e6.png)​

```c
#define  LIB_MEM_CFG_HEAP_SIZE          5u * 1024u     /* Configure heap memory size         [see Note #2a].           */
```

‍

当然，此时UCOS仍然是不能运行的，还需要再`Systick`​的中断函数中增加`OS_CPU_SysTickHandler`​函数，作为系统的“心脏”。

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694878595307-7571c493-78ff-4875-9dbb-ffd2275bca3a.png)​

然后修改下图文件的内容：

​`OS_CPU_PendSVHandler`​替换为`PendSV_Handler`​

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694881626083-2d82c309-2d4f-4e3b-a4f1-f0076b85981e.png)​

同时注释或者删掉`stm32f1xx_it.c`​文件中的`PendSV_Handler`​函数，否则编译器报错函数多重定义：

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694878655862-5796ecf6-b322-4bcc-bd70-96794dbf205d.png)​

然后再`main.c`​包含头文件`includes.h`​，并修改以下部分：

修改`<stm32f10x_lib.h>`​为`<stm32f1xx_hal.h>`​

​![](https://cdn.nlark.com/yuque/0/2023/png/35901054/1694879709352-ed1adf06-8ad8-4d3a-a4e0-a7b9a269302c.png)​

至此UCOSIII的移植就完成了。

---

 

# 5. **测试**

创建两个任务进行测试：

```c
/* USER CODE BEGIN PD */
/*任务优先级*/
#define START_TASK_PRIO			3
#define LED0_TASK_PRIO			4
#define LED1_TASK_PRIO			5


/*任务堆栈大小*/
#define START_STK_SIZE			128
#define LED0_STK_SIZE			128
#define LED1_STK_SIZE			128

/*任务堆栈*/
CPU_STK START_TASK_STK[START_STK_SIZE];
CPU_STK LED0_TASK_STK[LED0_STK_SIZE];
CPU_STK LED1_TASK_STK[LED1_STK_SIZE];


/*任务控制块*/
OS_TCB StartTaskTCB;
OS_TCB Led0TaskTCB;
OS_TCB Led1TaskTCB;


/*任务函数声明*/
void start_task(void *p_arg);
void led0_task(void *p_arg);
void led1_task(void *p_arg);
/* USER CODE END PD */
```

```c
int main(void)
{
  /* USER CODE BEGIN 1 */
	uint8_t ERROE = 0;
	OS_ERR err;
	CPU_SR_ALLOC();
	OSInit(&err);
  /* USER CODE END 1 */

  /* MCU Configuration--------------------------------------------------------*/

  /* Reset of all peripherals, Initializes the Flash interface and the Systick. */
  HAL_Init();

  /* USER CODE BEGIN Init */

  /* USER CODE END Init */

  /* Configure the system clock */
  SystemClock_Config();

  /* USER CODE BEGIN SysInit */

  /* USER CODE END SysInit */

  /* Initialize all configured peripherals */
  MX_GPIO_Init();
  /* USER CODE BEGIN 2 */

  	 /*进入临界区*/
   	OS_CRITICAL_ENTER();

    OSTaskCreate((OS_TCB		*)&StartTaskTCB,
                 (CPU_CHAR	*)"Start Task",
                 (OS_TASK_PTR  )start_task,
                 (void        *)0,
                 (OS_PRIO      )START_TASK_PRIO,
				 (CPU_STK     *)&START_TASK_STK[0],
				 (CPU_STK_SIZE )START_STK_SIZE/10,
				 (CPU_STK_SIZE )START_STK_SIZE,
				 (OS_MSG_QTY   )0,
				 (OS_TICK      )0,
				 (void        *)0,
				 (OS_OPT       )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
				 (OS_ERR      *)&err);

    /*退出临界区*/
    OS_CRITICAL_EXIT();
    OSStart(&err);
  /* USER CODE END 2 */

  /* Infinite loop */
  /* USER CODE BEGIN WHILE */
  while (1)
  {
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```

```c
/* USER CODE BEGIN 4 */
void start_task(void *p_arg){
	OS_ERR err;
	CPU_SR_ALLOC();
	p_arg = p_arg;

	/*进入临界区*/
	OS_CRITICAL_ENTER();

	OSTaskCreate((OS_TCB		*)&Led0TaskTCB,
			    (CPU_CHAR	*)"Led0 Task",
			    (OS_TASK_PTR  )led0_task,
			    (void        *)0,
			    (OS_PRIO      )LED0_TASK_PRIO,
			    (CPU_STK     *)&LED0_TASK_STK[0],
			    (CPU_STK_SIZE )LED0_STK_SIZE/10,
			    (CPU_STK_SIZE )LED0_STK_SIZE,
			    (OS_MSG_QTY   )0,
			    (OS_TICK      )0,
			    (void        *)0,
			    (OS_OPT        )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
			    (OS_ERR      *)&err);

	OSTaskCreate((OS_TCB		*)&Led1TaskTCB,
			    (CPU_CHAR	*)"Led1 Task",
			    (OS_TASK_PTR  )led1_task,
			    (void        *)0,
			    (OS_PRIO      )LED1_TASK_PRIO,
			    (CPU_STK     *)&LED1_TASK_STK[0],
			    (CPU_STK_SIZE )LED1_STK_SIZE/10,
			    (CPU_STK_SIZE )LED1_STK_SIZE,
			    (OS_MSG_QTY   )0,
			    (OS_TICK      )0,
			    (void        *)0,
			    (OS_OPT        )(OS_OPT_TASK_STK_CHK | OS_OPT_TASK_STK_CLR),
			    (OS_ERR      *)&err);

	/*退出临界区*/
	OS_CRITICAL_EXIT();
}

void led0_task(void *p_arg){
	OS_ERR err;
	while(1){
		led0_turn();
		OSTimeDly(200,OS_OPT_TIME_DLY,&err);
	}
}
void led1_task(void *p_arg){
	OS_ERR err;
	while(1){
		led1_turn();
		OSTimeDly(300,OS_OPT_TIME_DLY,&err);
	}
}
/* USER CODE END 4 */
```

最终移植圆满成功。
