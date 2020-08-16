## Makefile

三个模块：

​	目标：依赖（文件）

​	命令

Makefile的工作流程

* 第一阶段：构建关系树
* 第二阶段：自底向上执行命令

//根据文件的修改时间来决定是否重新执行命令

```makefile
all: add.c sub.c dive.c mul.c main.c
	gcc add.c sub.c dive.c mul.c main.c -o app
#命令前必须要有一个tab建


#新例子
app: add.o sub.o dive.o mul.o main.o
	gcc add.o sub.o dive.o mul.o main.o -o app

add.o:add.c
	gcc add.c -c
sub.o:sub.c
	gcc sub.c -c
mul.o:mul.c
	gcc mul.c -c
main.o:main.c
	gcc main.c -c
	
.PHONY:clean   
clean:
	-rm -f add.o
	-rm -f mul.o
	-@rm -f dive.o   #前面加上@符号代表不显示命令，只显示结果。加上-代表即使出错也继续往下进行命令
```

* 若执行目标没有依赖，则执行相应命令以示更新
* 默认条件下只执行第一棵关系树，若想执行其它目标，则需要自己指定，如`make mul.o`
* 正如前面所说，makefile通过修改时间来确定某个目标是否需要重新执行命令，如果我们指定了clean目标，当我们第一次使用make clean时是成功的，当我们再次使用时就会提示已经最新。此时就需要`.PHONY: xxx` 来指定xxx为一个伪命令