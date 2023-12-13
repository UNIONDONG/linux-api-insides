#  【Linux API 揭秘】container_of函数详解

> **Linux Version：6.6**
>
> **Author：Donge**
>
> **Github：[linux-api-insides](https://github.com/UNIONDONG/linux-api-insides)**

&nbsp;

## 1、container_of函数介绍

`container_of`可以说是内核中使用最为频繁的一个函数了，简单来说，它的主要作用就是根据我们结构体中的已知的成员变量的地址，来寻求该结构体的首地址，直接看图，更容易理解。

![image-20231212195328080](https://image-1305421143.cos.ap-nanjing.myqcloud.com/image/image-20231212195328080.png)

> 下面我们看看`linux`是如何实现的吧
>

## 2、container_of函数实现

```c
/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:	the pointer to the member.
 * @type:	the type of the container struct this is embedded in.
 * @member:	the name of the member within the struct.
 *
 * WARNING: any const qualifier of @ptr is lost.
 */
#define container_of(ptr, type, member) ({				\
	void *__mptr = (void *)(ptr);					\
	static_assert(__same_type(*(ptr), ((type *)0)->member) ||	\
		      __same_type(*(ptr), void),			\
		      "pointer type mismatch in container_of()");	\
	((type *)(__mptr - offsetof(type, member))); })

```

**函数名称**：`container_of`

**文件位置**：[include/linux/container_of.h](https://github.com/UNIONDONG/linux-api-insides/blob/main/include/linux/container_of.h)

该函数里面包括了一些封装好的宏定义以及函数，比如：`static_assert`、`__same_type`、`offsetof`，以及一些指针的特殊用法，比如：`(type *)0)`，下面我们一一拆解来看。

![image-20231213140920353](https://image-1305421143.cos.ap-nanjing.myqcloud.com/image/image-20231213140920353.png)

### 2.1 static_assert

```c
/**
 * static_assert - check integer constant expression at build time
 *
 * static_assert() is a wrapper for the C11 _Static_assert, with a
 * little macro magic to make the message optional (defaulting to the
 * stringification of the tested expression).
 *
 * Contrary to BUILD_BUG_ON(), static_assert() can be used at global
 * scope, but requires the expression to be an integer constant
 * expression (i.e., it is not enough that __builtin_constant_p() is
 * true for expr).
 *
 * Also note that BUILD_BUG_ON() fails the build if the condition is
 * true, while static_assert() fails the build if the expression is
 * false.
 */
#define static_assert(expr, ...) __static_assert(expr, ##__VA_ARGS__, #expr)
#define __static_assert(expr, msg, ...) _Static_assert(expr, msg)
```

**函数名称**：`static_assert`

**文件位置**：[include/linux/build_bug.h](https://github.com/UNIONDONG/linux-api-insides/blob/main/include/linux/build_bug.h)

**函数解析**：该宏定义主要用来 <font color = "red">**在编译时检查常量表达式，如果表达式为假，编译将失败，并打印传入的报错信息**</font>

- `expr`：该参数表示传入进来的常量表达式
- `...`：表示编译失败后，要打印的错误信息
- `_Static_assert`：`C11`中引入的关键字，用于判断表达式`expr`并打印错误信息`msg`。

在`container_of`函数中，主要用来断言判断

```c
	static_assert(
        __same_type(*(ptr), ((type *)0)->member)  ||   __same_type(*(ptr), void) ,
        "pointer type mismatch in container_of()"
	);
```

&nbsp;

### 2.2 __same_type

```c
/* Are two types/vars the same type (ignoring qualifiers)? */
#ifndef __same_type
# define __same_type(a, b) __builtin_types_compatible_p(typeof(a), typeof(b))
#endif
```

**函数名称**：`__same_type`

**文件位置**：[include/linux/compiler.h](https://github.com/UNIONDONG/linux-api-insides/blob/main/include/linux/compiler.h)

**函数解析**：<font color = "red">**该宏定义用于检查两个变量是否是同种类型**</font>

- `__builtin_types_compatible_p`：`gcc`的内建函数，判断两个参数的类型是否一致，如果是则返回1
- `typeof`：`gcc`的关键字，用于获取变量的类型信息

了解完`__same_type`，想要理解`__same_type(*(ptr), ((type *)0)->member)`，需要先弄明白`(type *)0`的含义。

&nbsp;

> 更多干货可见：[高级工程师聚集地](https://t.zsxq.com/0eUcTOhdO)，助力大家更上一层楼！

&nbsp;

### 2.3 (type *)0

`(type *)0`，该如何理解这个表达式呢？

- 首先，`type`是我们传入进来的结构体类型，比如上面讲到的`struct test`，而这里所做的<font color = "red">**可以理解为强制类型转换**</font>：`(struct test *)addr`。
- `addr`可以表示内存空间的任意的地址，我们在强制转换后，默认后面一片的内存空间存储的是该数据结构。

![image-20231213144714508](https://image-1305421143.cos.ap-nanjing.myqcloud.com/image/image-20231213144714508.png)

- 而`(type *)0`的作用，也就是默认将0地址处的内存空间，转换为该数据类型。

![image-20231213144912371](https://image-1305421143.cos.ap-nanjing.myqcloud.com/image/image-20231213144912371.png)

- 我们就把`0`，当作我们正常的`addr`地址变量来操作，`((type *)0)->member`，就是获取我们结构体的成员对象。
- `((type *)0)->member`：是一种常见的技巧，<font color="red" >**用于直接获取结构体`type`的成员`member`的类型，而不需要定义一个`type`类型的对象**</font>。

&nbsp;

### 2.4 offsetof

```c
#ifndef offsetof
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
#endif
```

**函数名称**：`offsetof`

**文件位置**：[include/linux/stddef.h](https://github.com/UNIONDONG/linux-api-insides/blob/main/include/linux/stddef.h)

**函数解析**：<font color = "red">**该宏定义用于获取结构体中指定的成员，距离该结构体偏移量。**</font>

![image-20231213152249395](https://image-1305421143.cos.ap-nanjing.myqcloud.com/image/image-20231213152249395.png)

- `TYPE`：表示结构体的类型
- `MEMBER`：表示指定的结构体成员
- `__builtin_offsetof`：`gcc`内置函数，直接返回偏移量。

&nbsp;

在新的`linux`源码中，直接引用了`gcc`内置的函数，而在老的内核源码中，该偏移量的实现方式如下：

```c
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

同样用到了`((TYPE *)addr)`，上面我们知道

- `((TYPE *)addr)->MEMBER`：表示获取该结构体的成员
- `&((TYPE *)addr)->MEMBER)`：加了一个`&`，表示地址，取该成员的内存地址。
  - 比如我们`addr=0x00000010`，那么`&((TYPE *)0x00000010)->MEMBER)`就相当于`0x00000010+size`
  - 比如我们`addr=0`，那么`&((TYPE *)0)->MEMBER)`就相当于`size`

&nbsp;

到这里，我们对`container_of`函数内部涉及的相关知识了然于胸，下面我们再来看`container_of`，简直容易到起飞。

&nbsp;

### 2.5 container_of

```c
#define container_of(ptr, type, member) ({				\
	void *__mptr = (void *)(ptr);					\
	static_assert(__same_type(*(ptr), ((type *)0)->member) ||	\
		      __same_type(*(ptr), void),			\
		      "pointer type mismatch in container_of()");	\
	((type *)(__mptr - offsetof(type, member))); })
```

- `static_assert`：断言信息，避免我们传入的参数类型不对，而做的编译检查处理，直接忽略。

```
#define container_of(ptr, type, member) ({				\
	void *__mptr = (void *)(ptr);					\
	((type *)(__mptr - offsetof(type, member))); })
```

- `offsetof(type, member)`：计算的是结构体中的成员的偏移量，这里称为`size`

- `(__mptr - offsetof(type, member))`：也就是根据我们已知的成员变量地址，计算出来结构体的首地址
- `((type *)(__mptr - offsetof(type, member)))`：最后强制转换为`(type *)`，结构体指针。

> 比如，我们已知的结构体成员的地址为`0xffff0000`，计算之后如下：

![image-20231213151416841](https://image-1305421143.cos.ap-nanjing.myqcloud.com/image/image-20231213151416841.png)

## 3、总结

`linux`内核中，小小的一个函数，内部包括的技巧如此之多：`static_assert`、`__same_type`、`(type *)0`、`offsetof`。

了解完内部完整的实现手法之后，我们也可以手码一个`container_of`了 :)

![image-20231119211155587](https://image-1305421143.cos.ap-nanjing.myqcloud.com/image/image-20231119211155587.png)

&nbsp;