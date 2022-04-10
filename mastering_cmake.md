# 4. 编写CMakeLists文件

## 4.2 Basic Commond

在工程中出现的每一个`project`命令， Cmake都会生成一个顶层文件，包含所有CmakeLists里面的target和由`add_subdirectory`添加的子目录。 如果`add_subdirectory`的`EXCLUDE_FROM_ALL`选项使用的话，生成的子项目就不会出现在顶层文件中，

## install， component关键字

### 安装时，component之间可能有依赖， 

headers component依赖libraries component，

```cmake
cpack_add_component(headers DEPENDS libraries)
```

### ldd命令

```
Getprerequisites.cmake, provides the get prercquisites function to analyze and
classify the prerequisite shared libraries upon which an executable depends
```



### Exporting and Importing Targets

将磁盘上的可执行文件，转变成logical target inside a cmake project， 

```cmake
add_executable (generator IMPORTED) 
set_property (TARGET generator PROPERTY IMPORTED LOCATION "/path/to/some generator")
add_custom_command (OUTPUT generated.c COMMAND generator generated.c)
add_executable (myexe srcl.c src2.c generated.c)
```

```
If COMMAND specifies an executable target (created by ADD_EXECUTABLE) it will automatically be replaced by the location of the executable created at build time. Additionally a target-level dependency will be added so that the executable target will be built before any target using this custom command. However this does NOT add a file-level dependency that would cause the custom command to re-run whenever the executable is recompiled.
```

#### Imported Target 作用域

```cmake
The scope of the definition of an IMPORTED target is the directory where it was defined. It may be accessed and used from subdirectories, but not from parent directories or sibling directories. The scope is similar to the scope of a cmake variable.

By default, the IMPORTED target name has scope in the directory in which it is created and below. We can use the GLOBAL option to extended visibility so that the target is accessible globally in the build system.
```



**add_custom_command 的command 可以是executable 的 target**， 

这种方式虽然方便，但是还需要知道可执行文件在磁盘上的位置，

### export函数

### add_custom_command on a Target

可以实现，build完之后copy到指定位置，实现安装

## Variables and Cache Entries

### 变量作用域

当一个子目录被创建，或者一个函数被调用，一个新的变量作用域就被创建了，是以当前作用域所有变量的值初始化的。

```cmake
function (foo)
  message ($(test))  # test is 1 here
  set(test 2)
  message ($(test)) # test is 2 here, but only in this scope
endfunction( )

set(test 1)
foo()
message ($(test))  # test is still 1 here
```

```cmake
# 变量只有一个值，也可以使用foreach，
set(RPM_TYPE a)
message("RPM_TYPE=${RPM_TYPE}")
foreach(item ${RPM_TYPE})
	message ( "Don't forget to buy one ${item}" )  # a
endforeach()

#可以直接在变量上做添加动作，添加后变成列表list
list(APPEND RPM_TYPE b)
list(APPEND RPM_TYPE c)
message("RPM_TYPE=${RPM_TYPE}")   #a;b;c

foreach(item ${RPM_TYPE})
	message ( "Don't forget to buy one ${item}" )
endforeach()
```

### 上层CMakeCache里的缓存变量，下层可以使用么？

```cmake
# 上层set的缓存变量， 下面的子目录中都可以使用。
# 缓存变量设置，相当于 user interface， 

# 缓存变量可以被CMakeLists文件里的set命令覆盖， 但是cache中的变量不变，
# 即使是set命令，Cache option也不能覆盖cache中的变量， 缓存中的变量一旦创建不可更改，只可能是"从外部"-D修改，
# In the rare event that you really want to change a cached variable's value you can use the
# force option in combination with the cAcHE option to the set command.
```

### 变量引号问题

```makefile
# CMP0054 
## 建议： if 判断两次引用变量不要加引号，
set(A E)
set(E "")

if("${A}" STREQUAL "")
  message("Result is TRUE before CMake 3.1 or when CMP0054 is OLD")
else()
  message("Result is FALSE in CMake 3.1 and above if CMP0054 is NEW")
endif()
```





## add_subdirectory

```cmake
使用 add_subdirectory 调用树构建项目的一个限制是，CMake不允许将 target_link_libraries 与定义在当前目录范围之外的目标一起使用。对于本示例来说，这不是问题。
在下一个示例中，我们将演示另一种方法，我们不使用add_subdirectory，而是使用 module include来组装不同的CMakeLists.txt文件，它允许我们链接到当前目录之外定义的目标
```

# Cmake的一些细节

* 每次cmake时-D带的变量会缓存到CMakeCache文件中，会追加



### 移动编出来的so的例子：

```cmake
set(CMAKE_INSTALL_PREFIX ${PROJECT_BINARY_DIR}/output)

#get_target_property(Add_loc add LOCATION)
#get_target_property(divide_loc divide LOCATION)

message("Add_loc=${Add_loc}")
message("CMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}")

## 	生成器表达式，替代get_target_property函数，
add_custom_command(
	TARGET add
	POST_BUILD
	COMMAND mkdir -p ${CMAKE_INSTALL_PREFIX} && mv $<TARGET_PROPERTY:add,LOCATION> ${CMAKE_INSTALL_PREFIX}
)
add_custom_command(
	TARGET divide
	POST_BUILD
	COMMAND mkdir -p ${CMAKE_INSTALL_PREFIX} && mv $<TARGET_PROPERTY:divide,LOCATION> ${CMAKE_INSTALL_PREFIX}
)
```



# Key Concepts

#### global generator and local generator 

```cmake
Every directory has a local generator that is responsible for generating the Makefiles or project files for that directory. 

All of the local generators share a common global generator that oversees the build process. 

Finally, the global generator is created and driven by the cmake class itself.
```

#### cmMakefile class

```cmake
One way to think of the cmMakefile class is as a structure that starts out initialized with a few variables from its parent directory, and is then filled in as the CMakeLists file is processed.
```

#### cmCommand

```cmake
Each command in CMake is implemented as a separate C++ class, and has two main parts.

The first part of a command is the InitialPass method.
The InitialPass method receives the arguments and the cmMakefile instance for the directory currently being processed, and then performs its operations.
The results of the command are always stored in the cmMakefile instance.

The FinalPass of a command is executed after all commands (for the entire CMake project) have had their InitialPass invoked. Most commands do not have a Finalpass, but in some rare cases a command must do something with global information that may not be available during the initial pass.
```

#### Makefiles

* 所有CMakeLists file被处理完之后，才由生成器创建makefile文件，

```cmake
Once all of the CMakeLists files have been processed the generators use the information
collected into the cmMakefile instances to produce the appropriate files for the target build
system (such as Makefiles).
```

#### 储存在cmMakefile instances中的items



# 0. LinuxC

## 0.1 计算机的体系和结构

1. 虚拟地址VA到物理地址PA转换， 怎么保证地址不冲突的？

## 0.2 x86汇编程序基础

### 0.2.1 最简单的汇编程序

int指令中的立即数0x80是一个参数。在Linux内核中，int $0x80这种异常称为系统调用（System Call）。

eax和ebx寄存器的值是传递给系统调用的两个参数，eax的值是系统调用号，1表示_exit系统调用，ebx的值则是传给_exit系统调用的参数，也就是退出状态。

不同的系统调用需要的参数个数也不同，比如有的需要ebx、ecx、edx三个寄存器的值做参数。 

### 0.2.2 x86的寄存器

ebp，esp用于维护函数调用的堆栈。

eip 是程序计数器。

eflags保存着计算过程中产生的标志位，包括第 3 节 “整数的加减运算”讲过的进位、溢出、零、负数四个标志位，在x86的文档中这几个标志位分别称为CF、OF、ZF、SF。

## 0.3 汇编和C之间的关系

### 0.3.1 函数调用

```C
int bar(int c, int d)
{ 
  int e = c + d; 
 return e;
}
int foo(int a, int b)
{ 
  return bar(a, b);
}
int main(void)
{
  foo(2, 3);
  return 0;
}
```

```c
call 80483aa
//call指令，两个作用， 把执行完该函数(80483aa地址的函数)的下一条指令地址压栈，然后设置eip去执行 80483aa
```

```C
// 函数执行完，返回的操作
leave  //这个指令是函数开头的push %ebp和mov %esp,%ebp的逆操作, 
ret    //它是call指令的逆操作，
```

* 参数压栈传递，并且是从右向左依次压栈。
* ebp总是指向栈帧的栈底。
* **返回值通过eax寄存器传递**

### 0.3.2 变量在内存的分布

static 关键字变量在函数内，static不影响作用域，只是说明分配时和全局变量一样，不是函数调用时才分配的。

#### 结构体和联合体

## 0.4 链接详解

### 0.4.1 多目标文件的链接

实际上链接的过程是由一个链接脚本（Linker Script）控制的，**链接脚本决定了给每个段分配什么地址，如何对齐，哪个段在前，哪个段在后，哪些段合并到同一个Segment**，另外链接脚本还要插入一些符号到最终生成的文件中，例如__bss_start、_edata、_end等。如果用ld做链接时没有用-T选项指定链接脚本，则使用ld的默认链接脚本，默认链接脚本可以用ld --verbose命令查看（由于比较长，只列出一些片断）：

### 0.4.2 extern和static关键字

extern可以写在函数内部，这样就只有块作用域，别的不影响，只是作用域不同

```C
// 函数声明中的extern关键字可以不写，下面是等价的
extern void push(char);
void push(char);
// 表示，push函数是具有External Linkage属性的，链接器会在链接的目标文件中寻找它的定义

// static关键字修饰一个函数声明，则表示该标识符具有Internal Linkage，编译之后，符号是LOCAL的，不参与链接
```

extern修饰变量一样，但是不能省略extern关键字。

如果一个模块内的变量和函数不想被外部模块使用，就声明为static的，这样符号是LOCAL的，不参与链接

### 0.4.3 静态库

链接器可以从静态库中只取出需要的部分来做链接

### 0.4.4 共享库

-fPIC 位置无关代码



### 0.4.5 虚拟内存管理

![image-20220110221213623](mastering-cmake.assets/image-20220110221213623.png)

## 0.5 地址无关代码

### 0.5.1 背景

```c++

```

### 0.5.2 装载时重定位

```python
# 之前的装载时重定位，并不能解决地址冲突问题，程序是整体加载的，指令和数据的相对位置不变， 
# 例如，一个程序在编译时假设被装载的目标地址是0x1000，但是装载时发现0x1000被使用了，从0x4000开始有空闲，则该程序就被装载到0x4000，程序指令或数据中的所有绝对引用只要加上0x3000偏移量即可。

# 装载时重定位不能解决共享对象so的问题
# 因为指令部分是在多个进程中共享的，由于装载时重定位会修改指令，所以没有办法做到同一份指令被多个进程共享，因为指令被重定位后对于每个进程是不同的。但是，可修改数据部分对于不同的进程来说有不同副本，所以可以采用上述装载时重定位方式解决，

# gcc， -shared 即是这种
```

### 0.5.3 地址无关代码

上述装载时重定位无法解决代码段共享问题，因此需要解决指令部分绝对地址的重定位问题，

我们**希望指令部分在装载时不需要因为装载地址的改变而改变**，所以实现的思路就是把指令中那些需要改变的部分分离出来，和数据部分放到一起，这样指令部分可以保持不变，这种方案即**地址无关代码（Position-independent Code）**。

先来分析模块中各种类型的地址引用方式。

* 模块内部函数调用，跳转等
* 模块内部数据访问，比如模块内定义的全局变量，静态变量
* 模块外部函数调用
* 模块外部数据访问，比如其他模块定义的全局变量， 本模块内extern声明的

#### 0.5.3.1 模块内部函数调用，跳转

```python
模块内部的跳转，函数调用都可以采用相对地址，所以这种指令不需要重定位
```

#### 0.5.3.2 模块内部数据访问

```python
任何一条指令和他要访问的数据之间的相对位置是固定的，那么只需要相对于当前指令加上相应偏移量就可以访问模块内数据了，
```

#### 0.5.3.3  模块间数据访问

```python
ELF在数据段里面建立一个指向其他模块全局变量的，全局偏移表(GOT)，链接器在装载模块时会查找每个来自其他模块的全局变量所在的地址，然后填充GOT中的每个项，由于GOT本身是放在数据段的，可以进行装载时重定向，
```

#### 0.5.3.4 模块外部函数调用

```python
和上面类似，只是GOT中的项保存的是目标函数的地址
```

### 0.5.4 小结

```python
extern int global;
inc foo()
{
  global=1;
}


# 编译.c文件时，无法根据上下文判断 extern int global 是定义在该模块的其他目标文件中还是定义在另外一个so中，既无法判断是否时跨模块访问。

# ELF共享库在编译时，默认都把定义在模块内部的全局变量当作是定义在其他模块的全局变量，即需要通过GOT访问。

# 当共享模块被装载时，如果某个全局变量在可执行文件中有副本，那么动态链接器就会把GOT中的地址指向该副本，这样该变量在运行时实际上只有一个实例，如果变量在共享模块中初始化，那么动态链接器还需要将该初始化值复制到可执行文件的变量副本，如果该全局变量在程序主模块中没有副本，那么GOT中的地址就指向模块内部的该变量副本
```



```python
gcc不使用-fPIC，那么就只使用装载时重定位， 不会产生地址无关代码，即不能被多个进程共享，失去了节省内存的优点。
但是，运行速度会快，因为省去了间接地址转换的计算过程，
```



# 1. 共享库依赖和模块的链接

## 1.1 模块的链接

这里模块指的是目标文件，.o文件

```cmake
目标文件的 .rel.text段， 重定位表，存放目标文件中的需要重定位的部分
```



```cmake
## 符号表，
链接过程的本质是把不同模块（目标文件）“粘合”在一起，在链接过程中，把函数和变量名称为符号（Symbol）.
每个目标文件，都有一个符号表，(Symbol Table)，记录了该目标文件用到的所有符号  (.symtab段)
nm simple.o 命令查看符号表
readelf -s simple.o 查看符号表，symtab段的信息
```

```cmake
# 符号表怎么看
ndx: 符号所在段
如果是定义在其他目标文件中，则是UND(SHN_UNDEF)
```

```cmake
### 动态链接和装载
Segment和Section是从不同角度划分同一个elf文件，链接视图和装载视图
因为目标文件不需要装载，因此没有程序头(Program Header Table)

## 动态链接
目标文件的链接过程推迟到运行时进行，假设Program1.o，Program2.o， Lib.o三个文件，
当我们运行Program1这个程序时，系统首先加载Program1.o，当系统发现Program1.o中用到了Lib.o，则去加载它，当所有依赖的目标文件被加载到内存，系统开始链接工作，最后把控制权交到Program1.o的程序入口，

## 动态链接使用的不是目标文件.o， 而是动态链接文件，即共享对象 .so
Program.c被编译成Program.o时，还不知道foobar函数的地址，当连接器把Program.o链接成可执行文件时，这时候连接器必须确定foobar函数的性质，
如果是定义在静态目标模块中的函数，那么按照静态链接法则，将foobar地址重定位，
如果是定义在否个so中的函数，那么会把它标记为动态链接符号，链接过程推迟到装载时。

## 查看进程虚拟地址空间
cat /proc/12985/maps

```



```cmake
# 可以看到.got段， LOAD的地址
objdump -h pic.so
# 得到需要动态链接时重定位项， 地址是在.got基础上加的，每4个字节是一项
objdump -R pic.so
```

## 1.2 延迟绑定(PLT)

```cmake
## 程序员的自我修养---->P200
```



## 1.3 动态链接具体实现

### 1.3.1 .interp段

可执行文件中的 .interp段中保存的字符串就是该可执行文件需要的动态链接器的路径

### 1.3.2 .dynamic段

保存了动态链接所需要的基本信息，比如依赖哪些so，动态链接符号表的地址，依赖的so的搜索路径等

### 1.3.3 动态符号表

```cmake
## 通常是 .dynsym段， 
很多时候动态链接的模块有两个表， .symtab 和 .dynsym ， 
 .symtab 保存了所有的符号，
 .dynsym 只保存了动态链接相关的符号
 
## 一些辅助的表， 
动态符号字符串表 .dynstr 
符号哈希表 .hash
```

### 1.3.4 动态链接重定位表

静态链接，导入符号是在链接时修正的。

对于动态链接，导入符号是在运行时才确定，

```cmake
# P735
static int a;
static int *p=&a;
# p的地址，绝对指向a，这种采用重定位方式，当动态链接器装载so时，进行重定位
```



```cmake
## LD_DEBUG变量，  P245
打开动态链接器的调试功能，


```

# 2. makefile

## 基础语法

```cmake
# make会自动选择那些受影响的源文件重新编译，不受影响的源文件则不重新编译 
1. make仍然尝试更新缺省目标，首先检查目标main是否需要更新，这就要检查三个条件main.o、stack.o和maze.o是否需要更新
2. make会进一步查找以这三个条件为目标的规则，然后发现main.o和maze.o需要更新，因为它们都有一个条件是maze.h，而这个文件的修改时间比main.o和maze.o晚，所以执行相应的命令更新main.o和maze.o。
3. 既然main的三个条件中有两个被更新过了，那么main也需要更新，所以执行命令gccmain.o stack.o maze.o -o main更新main

目标需要更新的情形，总结就是： 
## 目标没有生成
## 某个条件需要更新
## 某个条件的修改时间比目标晚

```



```cmake
语法： 
	命令前面加上@字符，则不显示命令，只执行，如 @echo "sss"， （因为echo本身会显示）
	通常make执行的命令如果出错（该命令的退出状态非0）就立刻终止，不再执行后续命令，但如果命令前面加了-号，即使这条命令出错，make也会继续执行后续命令。 （通常rm命令和mkdir命令前面要加-号）
```



```cmake
## 伪目标 .PHONY
.PHONY: clean
把clean当作一个特殊的名字使用，不管它存在不存在都要更新 

## 避免了如果存在clean文件时， make会认为clean不需要更新，（因为clean不依赖任何条件）
```

```cmake
makefile解析过程： Linux P349

```

##  隐含规则和模式规则

make -p 查看隐藏规则变量

```cmake
## 如果一个目标拆开写多条规则，其中只有一条规则允许有命令列表
## 如果一个目标在Makefile中的所有规则都没有命令列表，make会尝试在内建的隐含规则（Implicit Rule）数据库中查找适用的规则 (make的隐含规则数据库可以用make -p命令打印，打印出来的格式也是Makefile的格式，包括很多变量和规则)

## makefile变量类似c语言中的宏定义，是一串字符，在取值的地方展开。
```

```cmake
# default
OUTPUT_OPTION = -o $@
# default
CC = cc
# default
COMPILE.c = $(CC) $(CFLAGS) $(CPPFLAGS) $(TARGET_ARCH) -c
%.o: %.c
# commands to execute (built-in): 
	$(COMPILE.c) $(OUTPUT_OPTION) $<
	
$@和$<是两个特殊的变量，$@的取值为规则中的目标，$<的取值为规则中的第一个条件。
%.o:%.c是一种特殊的规则，称为模式规则（Pattern Rule）

例子：
	 makefile中，以main.o为目标的规则都没有命令列表，所以make会查找隐含规则，发现隐含规则中有这样一条模式规则适用，main.o符合%.o的模式，现在%就代表main（称为main.o这个名字的Stem），再替换到%.c中就是main.c。所以这条模式规则相当于
	 main.o: main.c 
	 					cc -c -o main.o main.c
```

## makefile变量

```cmake
## P354

特殊变量
$@，表示规则中的目标。
$<，表示规则中的第一个条件。
$?，表示规则中所有比目标新的条件，组成一个列表，以空格分隔。
$^，表示规则中的所有条件，组成一个列表，以空格分隔

例如：
  main: main.o stack.o maze.o 
        gcc main.o stack.o maze.o -o main
	可以改写成：
  	main: main.o stack.o maze.o 
  			 gcc $^ -o $@
			
```

2022/03/27

```makefile
include $(sources:.c=.d) 
%.d: %.c 
			set -e; rm -f $@; \ 
			$(CC) -MM $(CPPFLAGS) $< > $@.$$$$; \ 
			sed 's,\($*\)\.o[ :]*,\1.o $@ : ,g' < $@.$$$$ > $@; \ 
			rm -f $@.$$$$
```

```cmake
1. 
$(sources:.c=.d)是一个变量替换语法，把sources变量中每一项的.c替换成.d

include $(sources:.c=.d) 相当于
include main.d stack.d maze.d

# 类似于C语言的#include指示，这里的include表示包含三个文件main.d、stack.d和maze.d，这三个文件也应该符合Makefile的语法，
#一开始找不到.d文件，所以make会报警告。但是make会把include的文件名也当作目标来尝试更新，而这些目标适用模式规则%.d: %c，所以执行它的命令列表，

2. 在Makefile中$有特殊含义，如果要表示它的字面意思则需要写两个$, 两个$$转义成一个$

```



### 常用的make命令行选项



--------------------------------------3.29-------------------------------------

## makefile的prerequisite和add_custom_command

```cmake
## add_custom_command, in makefile terms this creates a new target in the following form:
OUTPUT: MAIN_DEPENDENCY DEPENDS
        COMMAND
        
## add_custom_target 的 DEPENDS 
Reference files and outputs of custom commands created with add_custom_command() command calls in the same directory (CMakeLists.txt file). 
***They will be brought up to date when the target is built.

## 如果add_custom_command不指定prerequisite， 如clean，那么如果touch clean，
## 则make 会认为OUTPUT不需要更新了，因为clean目标要是.PHONY的
```

```cmake
configure_file


```

--------------------------------------3.30-------------------------------------



```makefile
## This will always run both targets, 
## because some_file depends on other_file, which is never created.
## 无论 other_file 是否生成文件，都认为它已被更新

some_file: other_file
		touch some_file
	
other_file:
		echo "nothing"
```



```makefile
## When there are multiple targets for a rule, the commands will be run for each target
## $@ is an automatic variable that contains the target name.
用 $@ 表示当前的target，
all: f1.o f2.o

f1.o f2.o:
		echo $@
# Equivalent to:
# f1.o:
#	 echo f1.o
# f2.o:
#	 echo f2.o

```

Automatic Variables and Wildcards

```makefile
% Wildcard

## Static Pattern Rules
targets...: target-pattern: prereq-patterns ...
   commands
   
The essence is that the given target is matched by the target-pattern (via a % wildcard). Whatever was matched is called the stem. The stem is then substituted into the prereq-pattern, to generate the target's prereqs.

一个例子是： 编译.c文件到.o目标文件

objects = foo.o bar.o all.o
all: $(objects)

# These files compile via implicit rules
# Syntax - targets ...: target-pattern: prereq-patterns ...
# In the case of the first target, foo.o, the target-pattern matches foo.o and sets the "stem" to be "foo".
# It then replaces the '%' in prereq-patterns with that stem
$(objects): %.o: %.c

all.c:
	echo "int main() { return 0; }" > all.c

%.c:
	touch $@

clean:
	rm -f *.c *.o all
```

--------------------------------------3.31-------------------------------------

比较跳转指令

```assembly
start_loop: 
			cmpl $0, %eax
			je loop_exit
			
比较eax的值是不是0，如果是0就说明到达数组末尾了，就要跳出循环。
cmpl指令将两个操作数相减，但计算结果并不保存，只是根据计算结果改变eflags寄存器中的标志位。如果两个操作数相等，则计算结果为0，eflags中的ZF位置1。
je是一个条件跳转指令，它检查eflags中的ZF位，ZF位为1则发生跳转，ZF位为0则不跳转，继续执行下一条指令			
```



--------------------------------------4.03-------------------------------------

```c
## 汇编处理函数
You have only 7 usable registers, and one stack. 
Every function gets its arguments passed through the stack, and can return its return value through the %eax register. 
If every function modified every register, then your code will break, so every function has to ensure that the other registers are unmodified when it returns (other than %eax). You pass the arguments on the stack and your return value through %eax, so what should you do if need to use a register in your function? 
Easy: you keep a copy on the stack of any registers you’re going to modify so you can restore them at the end of your function. In the _add_a_and_b function, I did that for the %ebx register as you can see. 
For more complex function, it can get a lot more complicated than that, but let’s not get into that for now (for the curious: compilers will create what we call a “prologue” and an “epilogue” in each function
## 处理寄存器过程
In the prologue, you store the registers you’re going to modify, set up the %ebp (base pointer) register to point to the base of the stack when your function was entered, which allows you to access things without keeping track of the pushes/pops you do throughout the function, 
then in the epilogue, you pop the registers back, restore %esp to the value that was saved in %ebp, before you return).
```



llt切cmake项目阶段总结

```makefile
1. 编写makefile语法规则，有用的 make 命令行选项 -n，只输出命令不执行    （makefile tutorial github）
2. gdb调试， i sharedlibrary，显示程序依赖的动态库及其装载的路径   	（gdb 100个技巧）
    bt，调用栈
     si s n c等命令汇编单步，单步等
3. 过程中，cmake和make调试， make VERBOSE=1， cmake 传给连接器--Wl,-verbose
4. gcc动态链接， 和 编译时环境链接， LD_LIBRARY_PATH， LD_PRELOAD  （动态链接）

5. cmake 编译宏 target_compile_option
6. cmake output/release也能触发rebuild， 因为依赖output/release下面的头文件，
   cmake 生成的CmakeFiles文件夹下，有生成各个.o, .i, .s的 makefile，  
   build.cmake里面有cmake_check_buildsystem那个目标，
   后续可以看一下生成才到cmake文件， 把不懂的命令查一查， 像又一些自动生成的那些
```

# 3. 操作系统

## 3.1 通电开始

CPU的实模式和保护模式

```c++
1.实模式工作原理

实模式出现于早期8088CPU时期。当时由于CPU的性能有限，一共只有20位地址线（所以地址空间只有1MB），以及8个16位的通用寄存器，以及4个16位的段寄存器。所以为了能够通过这些16位的寄存器去构成20位的主存地址，必须采取一种特殊的方式。当某个指令想要访问某个内存地址时，它通常需要用下面的这种格式来表示：

　　(段基址：段偏移量)

　 其中第一个字段是段基址，它的值是由段寄存器提供的(一般来说，段寄存器有6种，分别为cs，ds，ss，es，fs，gs，这几种段寄存器都有自己的特殊意义，这里不做介绍)。

　 第二字段是段内偏移量，代表你要访问的这个内存地址距离这个段基址的偏移。它的值就是由通用寄存器来提供的，所以也是16位。那么两个16位的值如何组合成一个20位的地址呢？CPU采用的方式是把段寄存器所提供的段基址先向左移4位。这样就变成了一个20位的值，然后再与段偏移量相加。

即：

　　物理地址 = 段基址<<4 + 段内偏移

　　所以假设段寄存器中的值是0xff00，段偏移量为0x0110。则这个地址对应的真实物理地址是 0xff00<<4 + 0x0110 = 0xff110。

由上面的介绍可见，实模式的"实"更多地体现在其地址是真实的物理地址。
```

```c++
2.保护模式工作原理

随着CPU的发展，CPU的地址线的个数也从原来的20根变为现在的32根，所以可以访问的内存空间也从1MB变为现在4GB，寄存器的位数也变为32位。所以实模式下的内存地址计算方式就已经不再适合了。所以就引入了现在的保护模式，实现更大空间的，更灵活也更安全的内存访问。

在保护模式下，CPU的32条地址线全部有效，可寻址高达4G字节的物理地址空间; 但是我们的内存寻址方式还是得兼容老办法(这也是没办法的，有时候是为了方便，有时候是一种无奈)，即(段基址：段偏移量)的表示方式。当然此时CPU中的通用寄存器都要换成32位寄存器(除了段寄存器，原因后面再说)来保证寄存器能访问所有的4GB空间。

我们的偏移值和实模式下是一样的，就是变成了32位而已，而段值仍旧是存放在原来16位的段寄存器中，但是这些段寄存器存放的却不再是段基址了，毕竟之前说过实模式下寻址方式不安全，我们在保护模式下需要加一些限制，而这些限制可不是一个寄存器能够容纳的，于是我们把这些关于内存段的限制信息放在一个叫做全局描述符表(GDT)的结构里。全局描述符表中含有一个个表项，每一个表项称为段描述符。而段寄存器在保护模式下存放的便是相当于一个数组索引的东西，通过这个索引，可以找到对应的表项。段描述符存放了段基址、段界限、内存段类型属性(比如是数据段还是代码段,注意一个段描述符只能用来定义一个内存段)等许多属性,具体信息见下图：
```

![img](https://pic4.zhimg.com/80/v2-1a08d48367745c2870e8818b7881b373_1440w.jpg)



* X86 PC刚开机时， CPU处于实模式
* 开机时， CS=0xFFFF; IP=0xxxxx
* 寻址0xFFFF0（ROM BIOS映射区）
* 检查RAM，键盘，显示器，磁盘等
* 将磁盘0磁道0扇区512字节，即，引导扇区，读入到0x7c00处，
* 设置cs=0x7c0; IP=0x0000;



```assembly
## bootsect.s
rep movw   
## rep: 重复执行该语句直至寄存器cx为0
## 将DS：SI的内容送至ES：DI，note! 是复制过去，原来的代码还在

jmpi go, INITSEG
jmpi ## 段间跳转,  cs=INITSEG, ip=go

13号中断
10号中断
```

--------------------------------------4.06-------------------------------------

