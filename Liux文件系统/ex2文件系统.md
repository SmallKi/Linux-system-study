ｌｉｎｕｘ的用户列表文件在/etc/passwd,真正的密码文件是/etc/shadoe中

## ext2文件系统

在第一个块（ＢＯＯＴ　ＢＬＯＣＫ）中记录一共分了多少区，每个区多大，都安装的什么系统

ｂｏｏｔｂｌｏｃｋ的大小由ＰＣ联盟定义，１ｋ



后面的块才是由ｅｘｔ２文件系统接管，分成固定的若干块（４０９６Ｂ）

1block = 4096B

1block = 8磁盘扇区

1磁盘扇区=512B

1block = 32768bit



将若干块组合成一个组(group)

组的结构：

* 第一个块被称作超级块(super block)  保存了系统的信息，块的大小（每个组都有这样一个备份）
* 后面一组块称为ＧＤＴ（Group Descriptor table）块组描述符表
  * 记录块的使用情况
* 后面一个块(block bit map) 块位图，记录块的使用情况（块位图的大小决定了组的长度）
* 后面一个快(inode bit map)inode位图，记录文件的使用情况
* 后面若干块(inode table)　里面记录了一组ｉｎｏｄｅ记录，规定１２８Ｂ，其中６８Ｂ记录文件的属性，６０Ｂ记录数据块的指针
  * 数据块的指针最后三个是多级指针，倒数第三个为一级指针，倒数第二个为二级指针，倒数第三个为三级指针
  * 根目录的ｉｎｏｄｅ为２
  * ｅｘｔ４拓展了ｉｎｏｄｅ的长度，使得能指向的数据块更多
  * 文件夹也是一个ｉｎｏｄｅ，　当创建文件时，　会分配一个ｉｎｏｄｅ，　ｉｎｏｄｅ指向一个数据块，数据块中有一组记录，记录了文件名、ｉｎｏｄｅ、文件类型、记录长度
  * 数据块的编号是统一编号（１－最大）
* 最后是数据块





## 有inode有关的函数

#### utime\ chown \ truncate

#### access

作用：测试对某个文件是否有权限

```c++
#include<unistd.h>

int access(const char * path, int mode);
//mode的取值：
//F_OK 测试是否文件存在
//R_OK, 测试有没有读权限
//W_OK,测试有没有写权限
//X_OK, 测试是否有执行权限

成功(文件存在且请求的权限允许且没有出错)　返回０，否则返回－１
```





#### stat

作用：获取文件的磁盘信息

```C++
#include<sys/types.h>
#include<sys/stat.h>
#include<unistd.h>

int stat(const char *pathname, struct stat *buf);
//通过路径来获取信息，存入buf

int fstat(int fd, struct stat *buf);
//通过文件描述符来获取信息，存入ｂｕｆ

int lstat(const char *pathname, struct stat *buf);
//与ｓｔａｔ相同的功能，只是在链接文件上，它返回的是链接本身的信息，二ｓｔａｔ返回的是实际指向的文件的信息

           struct stat {
               dev_t     st_dev;         /* ID of device containing file */
               ino_t     st_ino;         /* inode number */
               mode_t    st_mode;        /* file type and mode */
               nlink_t   st_nlink;       /* number of hard links */
               uid_t     st_uid;         /* user ID of owner */
               gid_t     st_gid;         /* group ID of owner */
               dev_t     st_rdev;        /* device ID (if special file) */
               off_t     st_size;        /* total size, in bytes */
               blksize_t st_blksize;     /* blocksize for filesystem I/O */
               blkcnt_t  st_blocks;      /* number of 512B blocks allocated */

               /* Since Linux 2.6, the kernel supports nanosecond
                  precision for the following timestamp fields.
                  For the details before Linux 2.6, see NOTES. */

               struct timespec st_atim;  /* time of last access */
               struct timespec st_mtim;  /* time of last modification */
               struct timespec st_ctim;  /* time of last status change */

           #define st_atime st_atim.tv_sec      /* Backward compatibility */
           #define st_mtime st_mtim.tv_sec		//最近修改内容的时间
           #define st_ctime st_ctim.tv_sec		//最近修改ＩＮＯＤＥ中内容的时间
           };


```


#### chmod

作用：改变文件的访问权限

```c++
       int chmod(const char *pathname, mode_t mode);
       int fchmod(int fd, mode_t mode);
```





#### link

作用：创建链接

```c++
int link(const char *oldpath, const char *newpath);
//创建一个硬链接

int symlink(const char *target, const char *linkpath);
//创建一个软链接

int unlink(const char *pathname);
//删除一个链接
//如果这个链接是一个硬链接，且计数为１．但有进程打开了它，则将计数置为０，但不立即删除，而等进程用完后再进行删除
//可用于临时文件，进程用ｏｐｅｎ打开一个文件，然后删除它，则在进程关闭后，这个文件也将跟着删除
```



## 文件夹相关函数

#### mkdir

作用：创建文件夹

```c++
#include<fcntl.h>
#include<sys/stat.h>
#include<sys/types.h>
int mkdir(const char *pathname, mode_t mode);
//根据路径创建文件夹

int mkdirat(int dirfd, const char *pathname, mode_t mode);
//如果路径是相对路径，则从dirfd描述符代表的文件的所在的位置进行寻址
//如果路径是绝对路径，则忽略第一个参数
```



#### opendir

作用：打开一个文件夹

```c++
#include<dirent.h>
#include<sys/types.h>

DIR *opendir(const char *name);
DIR *fdopendir(int fd);

返回值：成功返回文件流指针，否则返回ＮＵＬＬ
```



#### readdir

作用：读取文件夹的信息

```c++
#include<dirent.h>

struct dirent *readdir(DIR *dirp);
//根据dirp遍历整个文件夹， 返回搜索到的记录， 当返回值为ＮＵＬＬ且errno没有变化， 则代表遍历结束



           struct dirent {
               ino_t          d_ino;       /* Inode number */
               off_t          d_off;       /* Not an offset; see below */
               unsigned short d_reclen;    /* Length of this record */
               unsigned char  d_type;      /* Type of file; not supported
                                              by all filesystem types */
               char           d_name[256]; /* Null-terminated filename */
           };

//d_type
DT_BLK      This is a block device.
DT_CHR      This is a character device.
DT_DIR      This is a directory.
DT_FIFO     This is a named pipe (FIFO).
DT_LNK      This is a symbolic link.
DT_REG      This is a regular file.
DT_SOCK     This is a UNIX domain socket.
DT_UNKNOWN  The file type could not be determined.
```



#### rewinddir

作用：重置文件夹流

```c++
#include <sys/types.h>

#include <dirent.h>

void rewinddir(DIR *dirp);
//将流指针重置到开头
```

####　telldir

作用：返回文件夹流指针的位置

```c++
long telldir(DIR *dirp);
```





## linux系统的VFX（Virtual File System）

每个进程维护着一个文件描述符表

每个进程开始默认创建三个文件描述符 STDIN_FILENO,  STDOUT_FILENO , STDERR_FILENO

每个文件描述符指向一个文件结构体

```c++ 
file_structure{
   	f_pos;    //读写指针 
    f_op;   // ====> 指向驱动层的file_operation ，里面有各种操作硬件的函数
    f_denty;  //=====> 指向文件系统中的函数，如chmod等
}
```

