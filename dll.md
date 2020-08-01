## gcc编译器

#### 常用参数

* `-c` 只编译，不链接为可执行文件
* `-o` output-filename 指定生成的文件名
* `-g` 符号调试工具（用于GDB的调试）
* `-O` 对程序进行优化编译、链接
* `-O2` 对程序进行更好的优化
* `-L<path>` 指定动态库路径
* `-l<name>` 指定库名称，如libx.o库的库名称就是x
* `-I<path>` 指定头文件目录，/usr/include不用包含，会自动搜索
* `-rdynamic` 指示连接器把所有符号都添加到动态符号表里， 



#### 编写动态库

**概要**

​	linux的动态库文件名为`libxxx.so`，xxx就是库名

1. 生成动态库

```shell
gcc -fPIC -shared -o <库文件名称>.so 源文件名称.c
//fPIC代表与位置无关，动态库的特性
//-shared 代表生成共享库文件而不是可执行文件
```

2. 生成链接文件

​	生成链接文件时，可通过 `-L<path> -lxxx.so`来包含这个库  

3. 调用共享库

   第一种做法是将这个动态库文件拷贝到 /lib 或 /usr/lib下，这样就能够链接执行了

   第二种做法是在编译时用-L显式指定库的所在位置

   第三种做法是更改LD_LIBRARY_PATH，将当前库文件路径添加到动态库搜索路径上去

4. 使用动态库

   

#### linux提供的动态库API

`#include<dlfcn.h>`

另外在编译时需要添加ld动态库 `-ldl`

1. ****

   dlopen

   ```c++
   void * dlopen(const char* filename, int flags);
   
   //filename: 库文件所在路径和文件名
   //flags必须包含以下两个之一
   //RTLD_LAZY 延迟加载
   //RTLD_NOW 立即加载
   //返回值
   //打开错误返回NULL， 成功返回库引用
   ```

   