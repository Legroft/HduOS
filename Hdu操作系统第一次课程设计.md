#### 课程设计要求

自定义系统调用，实现下列功能：

1. 本地存在两个文本文件：a.txt和b.txt;

2. 用户程序将a.txt的文件内容读出来，通过系统调用将a.txt中的内容追加到b.txt末尾。

3. 此处要求，b.txt文件在系统调用函数中打开，接收传递进来的字符串，并写入到b.txt中。

   注意：本地文件a.txt和b.tx均不小于4KB。

#### 设计思路

1. 用户程序中：

   自定义一个函数（使用stat()函数）来获取a.txt的文件大小，并调用该函数的返回值unsigned int作为字符串buf的长度（使用calloc实现动态char数组），调用linux的open(),read()等接口函数，将a.txt中的字符存到buf中，之后关闭文件，然后向系统调用传入buf长度以及buf字符串两个参数，最后释放内存。

2. 系统调用：

   接收从用户区传过来的两个参数，size（多少字节）和buf（字符串），通过对filp_open/filp_close /vfs_write等函数的调用，	将buf以append模式写入b.txt。


#### 源码

系统调用函数

```c
SYSCALL_DEFINE2(orange_test,unsigned long,size,char __user *,buf)
{
	char *p;
	p = (char*)kmalloc(sizeof(char)*size,GFP_KERNEL);
	if(!copy_from_user(p,buf,size))
	{
		struct file *fp;
   		mm_segment_t fs;
    		loff_t pos;
    		fp = filp_open("b.txt", O_WRONLY | O_APPEND, 0644);
    		if (IS_ERR(fp)) 
		{
        		printk("open file error\n");
        		return -1;
    		}
    		fs = get_fs();
    		set_fs(KERNEL_DS);
    		pos = 0;
    		vfs_write(fp, p,size, &pos);
    		filp_close(fp, NULL);
    		set_fs(fs);
	}
	kfree(p);
	return 0;
}
```

用户程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <syscall.h>
#define __NR_orange_test 339
unsigned long getFileSize(const char *path)
{
	unsigned long filesize = -1;
	struct stat statbuff;
	if(stat(path, &statbuff) < 0){
		return filesize;
	}else{
		filesize = statbuff.st_size;
	}
	return filesize;
}
int main()
{
    int fd =-1;
    char path[]="a.txt";
    unsigned int size=getFileSize(path);
    char *buf;
    buf=(char*)calloc(size,sizeof(char));
    fd=open(path,O_RDONLY);
    read(fd,buf,size);
    close(fd);
    syscall(__NR_orange_test,size,buf);
    free(buf);
    return 0;
}
```

#### 主要函数及解释说明

+ **stat()**：

  函数原型为：int stat(const char * file_name, struct stat *buf)。stat()用来将参数file_name 所指的文件状态, 复制到参数buf 所指的结构中。

  

+ **calloc()**：

  函数原型：void* calloc（unsigned int num，unsigned int size）。在内存的动态存储区中分配num个长度为size的连续空间，函数返回一个指向分配起始地址的指针；如果分配不成功，返回NULL。在动态分配完内存后，自动初始化该内存空间为零。一般使用后要使用 free(起始地址的指针) 对内存进行释放，不然内存申请过多会影响计算机的性能，以至于得重启电脑。如果使用过后不清零，还可以使用该指针对该块内存进行访问。

  

+ **open()**：

  打开某个指定的文件。函数原型为：int open(const char *pathname, int flags)。open()需要两个参数，第一个参数指定打开文件的路径，第二个参数指定文件的打开方式。其中flags参数必须选择O_RDONLY(只读)，O_WRONLY(只写)，O_RDWR(可读可写)的一种，还有一些参数是与以上三个以“按位或”的形式共同使用的，本次实验中只用到了O_APPEND(追加，只写入文件的末尾)。返回值是一个int 类型的文件描述符。

  

+ **read()**：

  函数原型为：size_t read(int fd, void *buf, size_t count)。第一个参数fd是需要读取的文件的描述符（opne()函数的返回值），第二个参数buf是想把文件读取到的内存的指针，第三个参数count 为读取的字节数。即把fd所指文件的count个字节放到buf所指的内存中。

  

+ **close()**：

  函数原型：int close(int fd)。对应open()，关闭fd所指的文件。

  

+ **copy_from_user()**：

  将数据从用户空间复制到内核空间。函数原型：unsigned long copy_from_user(void *to, const void __user *from, unsigned long n)。第一个参数to是内核空间的目的地址，第二个参数from是用户空间的源地址，第三个参数是数据的字节数。

  

+ **set_fs()/get_fs()**: 

  改变（获取）内核对内存地址检查的处理方式，其参数（或返回值）仅有两个取值：USER_DS（默认，对用户空间地址检查）、KERNEL_DS（对内核空间地址检查）。例如：一些函数含有__user修饰的参数，表示这是一个指向用户空间的，在内核中需要用内核空间代替时，就需要用set_fs()来做变换。

  

+ **filp_open()**:

   函数原型：struct file* filp_open(const char* filename, int open_mode, int mode)。第一个参数是文件的路径，第二个是文件打开方式（此处同open()函数），第三个参数在创建文件时设置文件读写权限，其他情况一般为0.返回值为一个文件指针。

  

+ **filp_close()**: 

  对应filp_open()，关闭文件。

  

+ **kmalloc()/kfree()**: 

  用于开辟（释放）一块内存空间。与平常使用的malloc()和free()函数类似。其中kmalloc()的函数原型为：void *kmalloc(size_t size, int flags)。size是需要开辟空间的大小，flags是分配标志，实验中用到的GFP_KERNEL是表示: 缺内存页的时候可以睡眠、允许启动磁盘IO、允许启动文件系统IO。

  

+ **vfs_write()**: 

  用于内核中文件的写操作。函数原型：ssize_t vfs_write(struct file* filp, const char __user* buffer, size_t len, loff_t* pos)。第一个参数是文件指针，第二个参数是需要写入的内容，第三个参数是写入的字节数，第四个参数为写入的位置。



#### 遇到问题及解决方案

##### 问题一：

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1588062417627.png)

##### 原因：

段错误，访问的内存超过了程序所给的空间。

##### 解决方案：

查找调试代码，对越界的地方进行修改。



##### 问题二：

警告 ISO C90 forbids mixed declarations and code

##### 原因：

变量定义之前任何一条非变量定义的语句都会引起这个警告。

##### 解决方案：

将非变量的定义移到变量定义之后。



##### 问题三：

从用户程序传入字符串到内核时，内核无法获取字符串的内容

##### 原因：

linux 操作系统和驱动程序运行在内核空间，应用程序运行在用户空间。两者不能简单地使用指针传递数据，所以我们考虑使用内核为驱动程序提供在内核空间和用户空间传递数据的方法，也就是copy_from_user函数。

##### 解决方案：

在我们自定义的系统调用中这样传入参数：SYSCALL_DEFINE2(orange_test,int,size,char __user *,buf);使用user来修饰传入的参数，让内核态可以访问用户内存。



##### 问题四：

在写入txt的过程中，遇到了只能写入8个字节的现象，比如从用户函数传入内核的字符串为“123456789”，写入txt的却只有“12345678”

##### 原因：

写入函数的参数调用出现了错误

修改前的部分关键源码为：

vfs_write(fp, p,sizeof(p), &pos);

分析一下这个函数调用的四个参数，分别是 文件指针，字符串指针，写入字节大小和写入的位置。我们经过讨论注意到，p是一个指针，而在64位操作系统中，一个指针的大小是占用8字节的，所以sizeof(p)为8，这也就解释了为什么只能写入8个字符。

##### 解决方案：

将关键的源码进行修改：

vfs_write(fp, p,size, &pos);

其中size是从用户程序中传过来的unsigned int值，表示a.txt中字节的大小



#### 分析程序运行结果

a.txt的内容如下，一共5153字节

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1588062995236.png)

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1588063016981.png)

b.txt中的内容如下，一共5347字节

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1588063058130.png)

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1588063068357.png)

运行程序后再次打开b.txt，可以看到a.txt中的内容已经被追加到b.txt中

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1588063143788.png)

此时b.txt的大小为5153 + 5347 = 10500 字节

![](https://photos-1256949929.cos.ap-shanghai.myqcloud.com/UTOOLS1588063167818.png)
