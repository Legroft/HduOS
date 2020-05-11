#### 课程设计要求

以内核模块编程实现下属功能：打开本地两个文本文件a,b，各自保存了任意给定的一个3X3矩阵(自然数），例如： 

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589030404296.png)

将这两个文件的矩阵读取出来，Matrix_A x Matrix_B 相乘之后的结果(3X3矩阵)存入另一个文本文件c中

要求能够对于随意输入的3X3矩阵，都能执行正确的计算，即a，b中的矩阵是可变的

#### 设计思路

使用kernel_read()读取文件，将文件内容写入到字符串中，再使用sscanf()函数将字符串中的内容格式化存储到3x3的数组中。

之后计算两个3x3矩阵的乘积，将计算后的结果存储到第三个矩阵中。

然后使用sprintf()将矩阵中的数据格式化存储到字符串中，最后使用kernel_write()将字符串写入文件。


#### 源码

myos2.c

```c
#include<linux/init.h>
#include<linux/module.h>
#include<linux/kernel.h>
#include <linux/module.h>
#include <linux/fs.h>
#include <linux/string.h>
#include <asm/uaccess.h>
#include <linux/slab.h>
static int hello_init(void){
	printk(KERN_ALERT"start process\n");
	int maxA[3][3]={0};
	int maxB[3][3]={0};
	int maxC[3][3]={0};
	char *pa,*pb,*pc;
	pa=(char*)kmalloc(sizeof(char)*100,GFP_KERNEL);
	pb=(char*)kmalloc(sizeof(char)*100,GFP_KERNEL);
	pc=(char*)kmalloc(sizeof(char)*100,GFP_KERNEL);
	struct file *fp_a;
	struct file *fp_b;
	struct file *fp_c;
   	mm_segment_t fs;
    	loff_t pos;
    	fp_a = filp_open("a.txt", O_RDWR, 0644);
	fp_b = filp_open("b.txt", O_RDWR, 0644);
	fp_c = filp_open("c.txt",O_WRONLY | O_APPEND, 0644);

    	if (IS_ERR(fp_a)) {
        	printk("open file a.txt error\n");
        	return -1;
    	}
	if (IS_ERR(fp_b)) {
        	printk("open file b.txt error\n");
        	return -1;
    	}
	if (IS_ERR(fp_c)) {
        	printk("open file c.txt error\n");
        	return -1;
    	}
    	fs = get_fs();
    	set_fs(KERNEL_DS);
    	pos = 0;
    	kernel_read(fp_a,pa,100, &pos);
	pos=0;
	kernel_read(fp_b,pb,100, &pos);
	
	sscanf(pa,"%d %d %d\n%d %d %d\n%d %d %d",&maxA[0][0],&maxA[0][1],&maxA[0][2],&maxA[1][0],&maxA[1][1],&maxA[1][2],&maxA[2][0],&maxA[2][1],&maxA[2][2]);
	
	sscanf(pb,"%d %d %d\n%d %d %d\n%d %d %d",&maxB[0][0],&maxB[0][1],&maxB[0][2],&maxB[1][0],&maxB[1][1],&maxB[1][2],&maxB[2][0],&maxB[2][1],&maxB[2][2]);
	
	int i;
	for(i=0;i<3;i++){
		int j;
		for(j=0;j<3;j++){
			int k;
			for(k=0;k<3;k++){
			maxC[i][j]=maxC[i][j]+maxA[i][k]*maxB[k][j];
			} 
		} 
	}
	
	sprintf(pc,"%d %d %d\n%d %d %d\n%d %d %d\n",maxC[0][0],maxC[0][1],maxC[0][2],maxC[1][0],maxC[1][1],maxC[1][2],maxC[2][0],maxC[2][1],maxC[2][2]);	
	
	int size=0;
	int m;
	for(m=0;m<3;m++){
		int n;
		for(n=0;n<3;n++){
			int tsize=0;
			while(maxC[m][n]>0)
			{
				tsize++;
				maxC[m][n]/=10;
	 		}
			size+=tsize+1;				
		}
	}
	pos=0;
	kernel_write(fp_c,pc,size,&pos);	
	printk(KERN_ALERT"completed\n");
    	filp_close(fp_a, NULL);
	filp_close(fp_b, NULL);
	filp_close(fp_c, NULL);
    	set_fs(fs);
	
	return 0;
}
static void hello_exit(void){
	printk(KERN_ALERT"goodbye\n");
}
module_init(hello_init);
module_exit(hello_exit);
MODULE_LICENSE("GPL");
```

Makefile

```
obj-m :=os2.o
os2-objs:=myos2.o
KDIR :=/usr/src/linux-5.6.2
PWD :=$(shell pwd)
default:
	make -C $(KDIR) M=$(PWD) modules
clean:
	make -C $(KDIR) M=$(PWD) clean
```

#### 主要函数及解释说明



+ **set_fs()/get_fs()**: 

  改变（获取）内核对内存地址检查的处理方式，其参数（或返回值）仅有两个取值：USER_DS（默认，对用户空间地址检查）、KERNEL_DS（对内核空间地址检查）。例如：一些函数含有__user修饰的参数，表示这是一个指向用户空间的，在内核中需要用内核空间代替时，就需要用set_fs()来做变换。

  

+ **filp_open()**:

   函数原型：struct file* filp_open(const char* filename, int open_mode, int mode)。第一个参数是文件的路径，第二个是文件打开方式（此处同open()函数），第三个参数在创建文件时设置文件读写权限，其他情况一般为0.返回值为一个文件指针。

  

+ **filp_close()**: 

  对应filp_open()，关闭文件。

  

+ **kmalloc()/kfree()**: 

  用于开辟（释放）一块内存空间。与平常使用的malloc()和free()函数类似。其中kmalloc()的函数原型为：void *kmalloc(size_t size, int flags)。size是需要开辟空间的大小，flags是分配标志，实验中用到的GFP_KERNEL是表示: 缺内存页的时候可以睡眠、允许启动磁盘IO、允许启动文件系统IO。

  

+ **vfs_read()**：
      ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
      参数中__user 表示buf是用户态指针，在内核中无法使用，因此需要设置使用环境，如下：
          mm_segment_t old_fs;
          old_fs = get_fs();
          set_fs(get_ds());
          vfs_read(file, (void __user *)addr, count, &pos);
          set_fs(old_fs);

  

+ **vfs_write()**: 

  用于内核中文件的写操作。函数原型：ssize_t vfs_write(struct file* filp, const char __user* buffer, size_t len, loff_t* pos)。第一个参数是文件指针，第二个参数是需要写入的内容，第三个参数是写入的字节数，第四个参数为写入的位置。
  
  
  
+ **sprintf()**:

  sprintf指的是字符串格式化命令，函数声明为 int sprintf(char *string, char *format [,argument,...]);，主要功能是把[格式化](https://baike.baidu.com/item/格式化/650)的数据写入某个字符串中，即发送格式化输出到 string 所指向的字符串。sprintf 是个[变参](https://baike.baidu.com/item/变参/9844833)函数。使用sprintf 对于写入buffer的字符数是没有限制的，这就存在了buffer溢出的可能性。解决这个问题，可以考虑使用 [snprintf](https://baike.baidu.com/item/snprintf)函数，该函数可对写入字符数做出限制。

  

+ **sscanf()**:

  sscanf 读取格式化的字符串中的数据。原型：int sscanf(const char *buffer,const char *format,[argument] ...  );
  
  参数：
  
  **buffer**  存储的数据
  
  **format**  窗体控件字符串。
  
  **argument**  可选自变量
  
  **locale**  要使用的区域设置



#### 遇到问题及解决方案

##### 问题一：

编译出错，出现 vfs_read[os2.ko] undefined !

##### 原因：

因为linux-4.0以后的版本取消了vfs_read()的符号导出EXPORT_SYMBOL(vfs_read)。

##### 解决方案：

使用int kernel_read(struct file *file, loff_t offset, char *addr, unsigned long count)函数

#### 实现过程

在Home目录下新建文件夹，命名为os2

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589032220516.png)

在文件夹中新建所需的文件

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589032272479.png)

a.txt,b.txt中内容均为下（c.txt为空）

1 2 3
4 5 6
7 8 9

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589032390510.png)

将源码写入myos2.c以及Makefile

然后终端中输入 make

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589032707807.png)

输入 sudo insmod ./os2.ko

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589032772912.png)

输入 lsmod 查看模块是否已经被加载

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589032918463.png)

输入 dmesg 查看日志输出

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589033002305.png)

查看文件内容是否正确，可以看到是两个矩阵相乘的结果

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589033056070.png)

输入 sudo rmmod os2 卸载模块

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589033129578.png)

输入 dmesg 查看日志

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1589033179387.png)
