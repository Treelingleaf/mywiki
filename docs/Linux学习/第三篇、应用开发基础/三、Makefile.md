# 三、Makefile

## 1. Makefile的引入和规则

给定一个目录：

```c
.
├── a.c
├── b.c
├── Makefile
# 其中a.c 依赖于 b.c
```

一般的编译：

```c
gcc -o test a.c b.c
```

> 这种办法编译的缺点就是，如果每次修改了其中一个文件，所有文件都得重新编译一遍

怎么解决：分别编译每个文件，然后再链接所有文件

```c
gcc -c -o a.o a.c
gcc -c -o b.o b.c
gcc -o test a.o b.o
```

怎么确定哪个文件被修改了:

比较a.c和a.o的时间，如果a.c的时间更新，代表a.c被修改了；比较test和a.o、b.o的时间，如果a.o或者b.o的时间更新，就需要重新链接生成新的test。总之，就是通过比较文件最后更新时间来确定文件有没有被修改。

Makefile就是通过上面的机制进行项目编译的。

---

‍

#### 1.1 Makefile规则

最最基本的规则：

```c
目标文件:依赖文件
	命令
```

> **解释：当依赖比目标新，就执行命令，依赖文件可能有一个，也可以有多个。**

把上面的例子写成Makefile：

```makefile
test:a.o b.o
	gcc -o test a.o b.o
a.o:a.c
	gcc -o a.o a.c
b.o:b.c
	gcc -o b.o b.c
```

> 上面的makefile包含三条规则，当要生成目标文件test的时候，如果发现a.o没有生成或者发现a.o的时间比test新，就会先生成a.o，往下寻找规则，发现a.o依赖于a.c，并执行命令生成a.o，makefile就是这样追根溯源的查找规则
>
> 当我修改了a.c后，我执行make，由a.o依赖a.c，并且a.c更新了，就会重新生成a.o，然后b.o依赖的b.c没有更新，不动，然后链接新的a.o和旧的b.o，生成新的test

‍

#### 1.2 Makefile的使用

使用很简单，直接适用命令：

```makefile
make [目标]
```

> 如果没指定目标，则默认第一个目标

## 2. Makefile的语法

#### 2.1 通配符

```makefile
%.o : %.c
	gcc -c -o $@ $<
```

>  **&quot;%&quot;是通配符，代表所有的&quot;.o&quot;和&quot;.c&quot;文件**
>
>  **&quot;$@&quot;代表规则中的目标**
>
>  **&quot;$&lt;&quot;代表规则中的第一个依赖**
>
>  **&quot;$^&quot;代表所有依赖**

‍

#### 2.2 假想目标 -- .PHONY

有这样一个情景，我想实现适用命令make clean 来清除编译产生的文件，实现如下：

```makefile
clean :
	rm *.o test
```

但是当我当前目录下有一个名为clean的文件的时候，就无法执行make clean了，因为规则里clean目标没有依赖文件，所以无法判断是否执行命令。

这个时候就得引入假想目标 .PHONY了，将clean定义为假想目标就不会存在上述的情况，就算当前目录存在同名文件，也能执行make clean的命令了。写法如下：

```makefile
clean :
	rm *.o test
.PHONY : clean
```

> **实现方法就是，不让系统再检查clean的文件是否存在或者依赖是否更新，而直接执行clean的指令。**

‍

#### 2.3 即时变量、延时变量、export

即时变量（又称简单变量），如

```makefile
A := xxx
# A的值即刻确定
```

延时变量，如：

```makefile
B = xxx
# B的值在B使用的时候才确认
```

例子：

```makefile
A := $(C)
B = $(C)
C = abc
all : 
	@echo $(A)
	@echo $(B)
# 结果就是A的值为空，B的值为abc，因为A是即时变量，即时将C赋值给了A，但是C是延时变量，没有使用C之前，C是空值，所以A就被赋为了空
```

> **涉及的知识：**
>
> 1. **使用变量的时候需要加上“$()”**
> 2. **变量不管定义在哪都是一样的效果，因为makefile是整篇解析，找不到的会向下查找**

四种赋值方式：

1. ​`:=`​ 即时赋值
2. ​`=`​ 延时赋值
3. ​`?=`​ 空赋值：如果是第一次定义才生效，如果已经定义过则忽略这次赋值
4. ​`+=`​ 追加赋值：他是即时变量还是延时变量，取决前面的定义

‍

## 3. Makefile函数

#### 3.1 foreach函数

​`foreach`​ 函数是GNU Make中的一个强大函数，用于在Makefile中执行循环迭代操作，通常用于处理列表中的元素。它的语法如下：

```make
$(foreach var, list, text)
```

* ​`var`​：循环中的临时变量，它会依次取`list`​中的每个元素的值。
* ​`list`​：一个以空格分隔的元素列表，可以是变量的值。
* ​`text`​：一个文本字符串，`var`​会逐个取`list`​中的元素值，并将`text`​中的`var`​替换为当前元素值。

> 理解：定义一个var临时变量，用于存储list中的单个元素，然后执行text的格式化操作，然后循环这一操作，直到list中没有元素了。

​`foreach`​ 函数的主要作用是将一个列表中的元素逐个进行操作，通常用于生成规则、文件名、目标列表等。

下面是一个示例，演示如何使用`foreach`​函数生成多个目标：

```make
SOURCE_FILES := file1.c file2.c file3.c
OBJECTS := $(foreach src,$(SOURCE_FILES),$(src:.c=.o))

all: $(OBJECTS)

%.o: %.c
    gcc -c $< -o $@

clean:
    rm -f $(OBJECTS)
```

在这个示例中，`$(OBJECTS)`​ 利用 `foreach`​ 函数迭代 `$(SOURCE_FILES)`​ 列表中的每个 `.c`​ 文件，并生成对应的 `.o`​ 文件名列表。然后，这些 `.o`​ 文件被用作`all`​目标的依赖项，从而创建了一个编译这些源文件的规则。

​`foreach`​ 函数是一个非常有用的工具，可用于各种情况，包括生成规则、目标列表、源文件列表等等，使Makefile更加灵活和自动化。

> $(src:.c=.o)，是将src变量的后缀.c替换为.o
>
> 还能这样追加$(src).o

‍

#### 3.2 filter函数和filter-out函数

​`filter`​ 是 GNU Make 中的一个函数，它用于从一个列表中筛选出满足条件的元素。`filter`​ 函数的基本语法如下：

```make
$(filter pattern, text)
```

* ​`pattern`​：是一个模式（pattern），通常是一个通配符表达式，用来匹配 `text`​ 中的元素。
* ​`text`​：是一个以空格分隔的元素列表，可以是变量的值。

​`filter`​ 函数会返回 `text`​ 中与 `pattern`​ **匹配**的元素列表。

​`filter-out`​和`filter`​恰恰相反，会返回`text`​ 中与 `pattern`​ **不匹配**的元素列表。

例如：

```makefile
SOURCE_FILES := a.c b.cpp c.o d.c e.cpp f.c g.o
C_SOURCE := $(filter %.c ,$(SOURCE_FILES))
CPP_SOURCE := $(filter %.cpp ,$(SOURCE_FILES))
OBJECTS := $(filter %.o ,$(SOURCE_FILES))
NO_C_SOURCE := $(filter-out %.c ,$(SOURCE_FILES))


all:
	@echo $(C_SOURCE)
	@echo $(CPP_SOURCE)
	@echo $(OBJECTS)
	@echo $(NO_C_SOURCE)
```

‍

#### 3.3 wilecard函数

​`wildcard`​ 是 GNU Make 中的一个函数，它用于查找文件系统中符合通配符模式的文件，并返回匹配的文件列表。`wildcard`​ 函数的基本语法如下：

```make
$(wildcard pattern)
```

* ​`pattern`​：是一个文件名通配符模式，可以包含 `*`​ 和 `?`​ 通配符，用来匹配文件名。

​`wildcard`​ 函数会返回匹配 `pattern`​ 的文件名列表。

以下是一个示例，演示如何使用 `wildcard`​ 函数：

```make
C_SOURCE_FILES := $(wildcard *.c)
```

在这个示例中，`$(wildcard *.c)`​ 将查找当前目录中所有以 `.c`​ 结尾的文件，并将它们的文件名列表赋给 `C_SOURCE_FILES`​ 变量。这通常用于自动发现源文件或其他文件，以便将它们包含在Makefile规则中。

​`wildcard`​ 函数非常有用，可以在Makefile中动态获取文件列表，从而使Makefile更具通用性和自动化性。

‍

#### 3.4 patsubst函数

​`patsubst`​ 是 GNU Make 中的一个函数，用于进行字符串替换操作，通常用于替换字符串中的一部分，使其符合特定的模式。`patsubst`​ 函数的基本语法如下：

```make
$(patsubst pattern,replacement,text)
```

* ​`pattern`​：是要匹配的模式，通常包含通配符，比如 `%`​，用于表示任意字符。
* ​`replacement`​：是替换 `pattern`​ 匹配的文本。
* ​`text`​：是要进行替换操作的文本。

​`patsubst`​ 函数会在 `text`​ 中查找 `pattern`​ 匹配的部分，并将它们替换为 `replacement`​。

以下是一个示例，演示如何使用 `patsubst`​ 函数：

```make
FILES := file1.txt file2.csv file3.txt
CSV_FILES := $(patsubst %.txt,%.csv,$(FILES))
```

在这个示例中，`$(patsubst %.txt,%.csv,$(FILES))`​ 将匹配 `FILES`​ 中以 `.txt`​ 结尾的文件名，然后将它们的扩展名替换为 `.csv`​。结果是 `CSV_FILES`​ 包含了 `file1.csv file2.csv file3.csv`​。

​`patsubst`​ 函数通常用于重命名文件、更改文件扩展名或进行类似的字符串替换操作，以根据需要修改文本。这在Makefile中非常有用，可以用于动态生成目标文件名或其他文本操作。

‍

## 4. 依赖文件

使用如下命令会生成对应的依赖文件：

```makefile
  gcc -M -MF b.d b.c
```

> 什么是依赖文件？长啥样
>
> 依赖文件就是Makefile的一个模式规则，代表b.o依赖于b.c ...........
>
> 这样一来，当Makefile里包含对应文件的依赖关系时，就可以在这些依赖被修改时重新编译这个文件，而不用手动的去添加依赖或者重新编译整个项目。

```makefile
b.o: b.c /usr/include/stdc-predef.h /usr/include/stdio.h \
 /usr/include/x86_64-linux-gnu/bits/libc-header-start.h \
 /usr/include/features.h /usr/include/x86_64-linux-gnu/sys/cdefs.h \
 /usr/include/x86_64-linux-gnu/bits/wordsize.h \
 /usr/include/x86_64-linux-gnu/bits/long-double.h \
 /usr/include/x86_64-linux-gnu/gnu/stubs.h \
 /usr/include/x86_64-linux-gnu/gnu/stubs-64.h \
 /usr/lib/gcc/x86_64-linux-gnu/9/include/stddef.h \
 /usr/lib/gcc/x86_64-linux-gnu/9/include/stdarg.h \
 /usr/include/x86_64-linux-gnu/bits/types.h \
 /usr/include/x86_64-linux-gnu/bits/timesize.h \
 /usr/include/x86_64-linux-gnu/bits/typesizes.h \
 /usr/include/x86_64-linux-gnu/bits/time64.h \
 /usr/include/x86_64-linux-gnu/bits/types/__fpos_t.h \
 /usr/include/x86_64-linux-gnu/bits/types/__mbstate_t.h \
 /usr/include/x86_64-linux-gnu/bits/types/__fpos64_t.h \
 /usr/include/x86_64-linux-gnu/bits/types/__FILE.h \
 /usr/include/x86_64-linux-gnu/bits/types/FILE.h \
 /usr/include/x86_64-linux-gnu/bits/types/struct_FILE.h \
 /usr/include/x86_64-linux-gnu/bits/stdio_lim.h \
 /usr/include/x86_64-linux-gnu/bits/sys_errlist.h
```

依赖关系的自动生成和清理规则：

```makefile
文件目录：
.
├── a.c
├── b.c
├── b.d
├── c.c
├── clean
├── include
│   └── c.h
└── Makefile
```

Makefile：

```makefile
objs = a.o b.o c.o

# 根据所有的.o文件，生成了依赖关系文件的列表
dep_files := $(patsubst %,.%.d, $(objs))
# 确保只包含已经存在的依赖关系文件
dep_files := $(wildcard $(dep_files))

# 定义了编译选项，包括警告错误（-Werror）和包含路径（-Iinclude）
CFLAGS = -Werror -Iinclude

test: $(objs)
        gcc -o test $^

# 如果 dep_files 不为空，那么它会执行下面的 include 语句
ifneq ($(dep_files),)
include $(dep_files) # 这里将包含所有依赖关系文件，这些文件列出了源文件和它们的依赖关系，以确保在头文件修改时源文件被重新编译
endif

%.o : %.c
        gcc $(CFLAGS) -c -o $@ $< -MD -MF .$@.d

clean:
        rm *.o test

distclean:
        rm $(dep_files)

.PHONY: clean
```

> 上面用到了include语法：

​`include`​ 是Makefile的一种关键语法。它用于在Makefile中包含其他文件的内容，通常用于引入其他Makefile片段、变量定义、规则等。

语法如下：

```make
include filename
```

* ​`filename`​ 是要包含的文件的名称，通常是相对或绝对路径。

​`include`​ 语句允许你将其他Makefile文件的内容合并到当前Makefile中，以便重用、组织和管理构建规则。这在大型项目中特别有用，因为它允许你将构建系统拆分为多个模块，并通过 `include`​ 来组合它们。

通常，`include`​ 语句用于包含包含变量、规则、配置选项等内容的外部文件，以便使Makefile更加模块化和易于维护。这有助于减少重复工作，提高Makefile的可读性和可维护性。

‍

## 5. 通用Makefile

文件结构：

```makefile
.
├── a
│   ├── Makefile
│   ├── sub2.c
│   └── sub3.c
├── include
│   ├── sub2.h
│   ├── sub3.h
│   └── sub.h
├── main.c
├── Makefile
├── Makefile.build
└── sub.c
```

顶层目录的makefile：

```makefile
#--------------------------------------------------------------------------------------#
# 配置头文件路径 例如：$(shell pwd)/other/include $(shell pwd)/include
include_path := 

# 配置源文件路径 例如：source/ other/source/
source_path := 

# 指定编译器的前缀 例如： arm-buildroot-linux-gnueabihf-
CROSS_COMPILE = 

# 编译选项，用于整个工程
CFLAGS := -Wall -O2 -g

#配置当前目录下要排除编译的文件 例如：sub.o
exclude-files := 
#--------------------------------------------------------------------------------------#

AS		= $(CROSS_COMPILE)as
LD		= $(CROSS_COMPILE)ld
CC		= $(CROSS_COMPILE)gcc
CPP		= $(CC) -E
AR		= $(CROSS_COMPILE)ar
NM		= $(CROSS_COMPILE)nm

STRIP		= $(CROSS_COMPILE)strip
OBJCOPY		= $(CROSS_COMPILE)objcopy
OBJDUMP		= $(CROSS_COMPILE)objdump

export AS LD CC CPP AR NM
export STRIP OBJCOPY OBJDUMP


# 编译时使用指定头文件路径选项
CFLAGS += $(foreach path, $(include_path), -I$(path))

# 指定库，如果有的话
LDFLAGS := 

export CFLAGS LDFLAGS

TOPDIR := $(shell pwd)
export TOPDIR

# 输出程序
TARGET := myapp

# 指定参加编译的文件或者目录
obj-y += $(patsubst %.c,%.o,$(wildcard *.c))
# 列出顶层目录要排除编译的文件
exclude-files := 
# # 从obj-y中移除要排除的文件
obj-y := $(filter-out $(patsubst %.c,%.o,$(exclude-files)),$(obj-y))
obj-y += $(source_path)



all : start_recursive_build $(TARGET)
	@echo $(TARGET) has been built!

start_recursive_build:
	make -C ./ -f $(TOPDIR)/Makefile.build

$(TARGET) : start_recursive_build
	$(CC) -o $(TARGET) built-in.o $(LDFLAGS)

clean:
	rm -f $(shell find -name "*.o")
	rm -f $(TARGET)

distclean:
	rm -f $(shell find -name "*.o")
	rm -f $(shell find -name "*.d")
	rm -f $(TARGET)

run : all
	@clear
	@./$(TARGET)
```

子目录a下的makefile：

```makefile
EXTRA_CFLAGS := -D DEBUG
CFLAGS_sub3.o := -D DEBUG_SUB3

obj-y += sub2.o
obj-y += sub3.o
＃ 可以改进一下 自动获取目标文件列表
```

> 改进子目录下的makefile：

```makefile
# 配置当前目录的编译选项，适用当前目录下的文件 例如：-D DEBUG
EXTRA_CFLAGS := 

# 列出当前目录要排除编译的文件 例如：sub.o
exclude-files := 

obj-y += $(patsubst %.c,%.o,$(wildcard *.c))

# 从obj-y中移除要排除的文件
obj-y := $(filter-out $(patsubst %.c,%.o,$(exclude-files)),$(obj-y))
```

Makefile.build

```makefile
PHONY := __build
__build:


obj-y :=
subdir-y :=
EXTRA_CFLAGS :=

include Makefile

# obj-y := a.o b.o c/ d/
# $(filter %/, $(obj-y))   : c/ d/
# __subdir-y  : c d
# subdir-y    : c d
__subdir-y	:= $(patsubst %/,%,$(filter %/, $(obj-y)))
subdir-y	+= $(__subdir-y)

# c/built-in.o d/built-in.o
subdir_objs := $(foreach f,$(subdir-y),$(f)/built-in.o)

# a.o b.o
cur_objs := $(filter-out %/, $(obj-y))
dep_files := $(foreach f,$(cur_objs),.$(f).d)
dep_files := $(wildcard $(dep_files))

ifneq ($(dep_files),)
  include $(dep_files)
endif


PHONY += $(subdir-y)


__build : $(subdir-y) built-in.o

$(subdir-y):
	make -C $@ -f $(TOPDIR)/Makefile.build

built-in.o : $(subdir-y) $(cur_objs)
	$(LD) -r -o $@ $(cur_objs) $(subdir_objs)

dep_file = .$@.d

%.o : %.c
	$(CC) $(CFLAGS) $(EXTRA_CFLAGS) $(CFLAGS_$@) -Wp,-MD,$(dep_file) -c -o $@ $<

.PHONY : $(PHONY)

```

‍
