# 【Linux API 揭秘】module_init与module_exit

> **Linux Version：6.6**
>
> **Author：Donge**
>
> **Github：linux-api-insides**
>
> 优化阅读体验：[传送门](https://uniondong.github.io/post//linux-api-揭秘/linux-api-揭秘module_init与module_exit)

&nbsp;

## 1、函数作用

`module_init`和`module_exit`是驱动中最常用的两个接口，主要用来注册、注销设备驱动程序。

并且这两个接口的实现机制是一样的，我们先以`module_init`为切入点分析。

&nbsp;

## 2、module\_init函数解析

### 2.1 module\_init

```c
#ifndef MODULE
/**
 * module_init() - driver initialization entry point
 * @x: function to be run at kernel boot time or module insertion
 *
 * module_init() will either be called during do_initcalls() (if
 * builtin) or at module insertion time (if a module).  There can only
 * be one per module.
 */
#define module_init(x)	__initcall(x);

......

#else /* MODULE */

......
    
/* Each module must use one module_init(). */
#define module_init(initfn)					\
    static inline initcall_t __maybe_unused __inittest(void)		\
    { return initfn; }					\
    int init_module(void) __copy(initfn)			\
        __attribute__((alias(#initfn)));		\
    ___ADDRESSABLE(init_module, __initdata);

......

#endif
```

**函数名称**：`module_init`

**文件位置**：[include/linux/module.h](https://github.com/UNIONDONG/linux-api-insides/blob/main/include/linux/module.h)

**函数解析**：

> 在`Linux`内核中，驱动程序可以以两种方式存在：内建`(Builtin)`和模块`(Module)`。内建驱动就是在编译时，直接编译进内核镜像中；而模块驱动则是在内核运行过程中动态加载卸载的。

`module_init`函数的定义位置有两处，使用`MODULE`宏作为判断依据。`MODULE`是一个预处理器宏，仅当该驱动作为模块驱动时，编译的时候会加入`MODULE`的定义。

> 这里难免会有疑问：为什么会有两套实现呢？

其实，当模块被编译进内核时，代码是存放在内存的`.init`字段，该字段在内核代码初始化后，就会被释放掉了，所以当可动态加载模块需要加载时，就需要重新定义了。

&nbsp;

#### 2.1.1 模块方式

当驱动作为可加载模块时，`MODULE`宏被定义，我们简单分析一下相关代码

```c
#define module_init(initfn)					\
    static inline initcall_t __maybe_unused __inittest(void)		\
    { return initfn; }					\
    int init_module(void) __copy(initfn)			\
        __attribute__((alias(#initfn)));		\
    ___ADDRESSABLE(init_module, __initdata);
```

- `static inline initcall_t __maybe_unused __inittest(void) { return initfn; }`：一个内联函数，返回传入的`initfn`函数。
    - `__maybe_unused` ：编译器指令，用于告诉编译器，该函数可能不会使用，以避免编译器产生警告信息。
- `int init_module(void) __copy(initfn) __attribute__((alias(#initfn)));`：`init_module`函数的声明
    - `__copy(initfn)`：编译器指令，也就是将我们的`initfn`函数代码复制到`init_module`中，
    - `__attribute__((alias(#initfn)))`：编译器指令，将`init_module`函数符号的别名设置为`initfn`。
- \_\_`_ADDRESSABLE(init_module, __initdata);`：一个宏定义，主要用于将`init_module`函数的地址放入`__initdata`段，这样，当模块被加载时，`init_module`函数的地址就可以被找到并调用。

**总的来说，如果是可加载的`ko`模块，`module_init`宏主要定义了`init_module`函数，并且将该函数与`initfn`函数关联起来，使得当模块被加载时，初始化函数可以被正确地调用。**

&nbsp;

#### 2.1.2 内建方式

当模块编译进内核时，`MODULE`宏未被定义，所以走下面流程

```c
#define module_init(x)	__initcall(x);
```

&nbsp;

### 2.2 \_\_initcall

```c
#define __initcall(fn) device_initcall(fn)

#define device_initcall(fn)		__define_initcall(fn, 6)

#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)

#define ___define_initcall(fn, id, __sec)			\
    __unique_initcall(fn, id, __sec, __initcall_id(fn))

#define __unique_initcall(fn, id, __sec, __iid)			\
    ____define_initcall(fn,					\
        __initcall_stub(fn, __iid, id),			\
        __initcall_name(initcall, __iid, id),		\
        __initcall_section(__sec, __iid))

#define ____define_initcall(fn, __unused, __name, __sec)	\
    static initcall_t __name __used 			\
        __attribute__((__section__(__sec))) = fn;

#define __initcall_stub(fn, __iid, id)	fn

/* Format: <modname>__<counter>_<line>_<fn> */
#define __initcall_id(fn)					\
    __PASTE(__KBUILD_MODNAME,				\
    __PASTE(__,						\
    __PASTE(__COUNTER__,					\
    __PASTE(_,						\
    __PASTE(__LINE__,					\
    __PASTE(_, fn))))))

/* Format: __<prefix>__<iid><id> */
#define __initcall_name(prefix, __iid, id)			\
    __PASTE(__,						\
    __PASTE(prefix,						\
    __PASTE(__,						\
    __PASTE(__iid, id))))

#define __initcall_section(__sec, __iid)			\
    #__sec ".init"

/* Indirect macros required for expanded argument pasting, eg. __LINE__. */
#define ___PASTE(a,b) a##b
#define __PASTE(a,b) ___PASTE(a,b)
```

**函数名称**：`__initcall`

**文件位置**：[include/linux/init.h](https://github.com/UNIONDONG/linux-api-insides/blob/main/include/linux/init.h)

**函数解析**：设备驱动初始化函数

&nbsp;

#### 2.2.1 代码调用流程

```c
module_init(fn)
    |--> __initcall(fn)
        |--> device_initcall(fn)
            |--> __define_initcall(fn, 6)
                |--> ___define_initcall(fn, id, __sec)
                    |--> __initcall_id(fn)
                    |--> __unique_initcall(fn, id, __sec, __iid)
                        |--> ____define_initcall(fn, __unused, __name, __sec)
                            |--> __initcall_stub(fn, __iid, id)
                            |--> __initcall_name(prefix, __iid, id)
                            |--> __initcall_section(__sec, __iid)
                        |--> ____define_initcall(fn, __unused, __name, __sec)
```

&nbsp;

> 进行函数分析前，我们先要明白`#`和`##`的概念

#### 2.2.2 #和##的作用

| 符号  | 作用  | 举例  |
| --- | --- | --- |
| ##  | `##`符号 可以是连接的意思 | 例如 `__initcall_##fn##id` 为`__initcall_fnid`那么，`fn = test_init`，`id = 6`时，`__initcall##fn##id` 为 `__initcall_test_init6` |
| #   | `#`符号 可以是**字符串化的意思** | 例如 `#id` 为 `"id"`，`id=6` 时，`#id` 为`"6"` |

&nbsp;

> 更多干货可见：[高级工程师聚集地](https://t.zsxq.com/0eUcTOhdO)，助力大家更上一层楼！

&nbsp;

#### 2.2.3 函数解析

> 下面分析理解比较有难度的函数

```c
#define device_initcall(fn)		__define_initcall(fn, 6)
#define __define_initcall(fn, id) ___define_initcall(fn, id, .initcall##id)
```

- `.initcall##id`：通过`##`来拼接两个字符串：`.initcall6`

```c
#define ___define_initcall(fn, id, __sec)			\
    __unique_initcall(fn, id, __sec, __initcall_id(fn))

/* Format: <modname>__<counter>_<line>_<fn> */
#define __initcall_id(fn)					\
    __PASTE(__KBUILD_MODNAME,				\
    __PASTE(__,						\
    __PASTE(__COUNTER__,					\
    __PASTE(_,						\
    __PASTE(__LINE__,					\
    __PASTE(_, fn))))))

/* Indirect macros required for expanded argument pasting, eg. __LINE__. */
#define ___PASTE(a,b) a##b
#define __PASTE(a,b) ___PASTE(a,b)
```

- `___PASTE`：拼接两个字符串
- `__initcall_id`：**它用于生成一个唯一的标识符，这个标识符用于标记初始化函数**。
    - `__KBUILD_MODNAME`：当前正在编译的模块的名称
    - `__COUNTER__`：一个每次使用都会递增计数器，用于确保生成名称的唯一性
    - `__LINE__`：当前代码的行号

&nbsp;

```c
#define __unique_initcall(fn, id, __sec, __iid)			\
    ____define_initcall(fn,					\
        __initcall_stub(fn, __iid, id),			\
        __initcall_name(initcall, __iid, id),		\
        __initcall_section(__sec, __iid))

#define ____define_initcall(fn, __unused, __name, __sec)	\
    static initcall_t __name __used 			\
        __attribute__((__section__(__sec))) = fn;

#define __initcall_stub(fn, __iid, id)	fn

/* Format: __<prefix>__<iid><id> */
#define __initcall_name(prefix, __iid, id)			\
    __PASTE(__,						\
    __PASTE(prefix,						\
    __PASTE(__,						\
    __PASTE(__iid, id))))

#define __initcall_section(__sec, __iid)			\
    #__sec ".init"
```

`__unique_initcall`：调用`____define_initcall`，关键实现部分

`____define_initcall`：定义一个名为 `__name` 的 `initcall_t` 类型的静态变量，并将其初始化为 `fn`，并放入特定的`__sec`段中。

- `__initcall_stub`：表示唯一的函数名`fn`
- `__initcall_name`：表示一个唯一的变量名
- `__initcall_section`： 生成一个唯一的段名。
- `#__sec ".init"`：将两个字符串拼接起来，比如：`__sec=.initcall6`，拼接后的段为：`.initcall6.init`，该段为最终存储的段。

&nbsp;

**字段通过链接器链接起来，形成一个列表进行统一管理。**

> 这些字段我们可以在`arch/arm/kernel/vmlinux.lds`中查看。

```c
......
__initcall6_start = .; KEEP(*(.initcall6.init)) KEEP(*(.initcall6s.init)) 
......
```

&nbsp;

## 3、module\_exit函数解析

> `module_exit`和`module_init`的实现机制几乎没有差别，下面就简单介绍一下。

### 3.1 module\_exit

```c
#ifndef MODULE

/**
 * module_exit() - driver exit entry point
 * @x: function to be run when driver is removed
 *
 * module_exit() will wrap the driver clean-up code
 * with cleanup_module() when used with rmmod when
 * the driver is a module.  If the driver is statically
 * compiled into the kernel, module_exit() has no effect.
 * There can only be one per module.
 */
#define module_exit(x)	__exitcall(x);

......

#else /* MODULE */

......
    
/* This is only required if you want to be unloadable. */
#define module_exit(exitfn)					\
    static inline exitcall_t __maybe_unused __exittest(void)		\
    { return exitfn; }					\
    void cleanup_module(void) __copy(exitfn)		\
        __attribute__((alias(#exitfn)));		\
    ___ADDRESSABLE(cleanup_module, __exitdata);

......

#endif
```

**函数名称**：`module_exit`

**文件位置**：[include/linux/module.h](https://github.com/UNIONDONG/linux-api-insides/blob/main/include/linux/module.h)

#### 3.1.1 模块方式

作为模块方式，与`module_init`的实现方式一样，定义`cleanup_module`与`exitfn`函数相关联，存放在`__exitdata`段内。

&nbsp;

#### 3.1.2 内建方式

当模块编译进内核时，`MODULE`宏未被定义，所以走下面流程

```c
#define module_exit(x)	__exitcall(x);
```

&nbsp;

### 3.2 \_\_exitcall

```c
#define __exitcall(fn)						\
    static exitcall_t __exitcall_##fn __exit_call = fn

#define __exit_call	__used __section(".exitcall.exit")
```

**函数名称**：`__initcall`

**文件位置**：[include/linux/init.h](https://github.com/UNIONDONG/linux-api-insides/blob/main/include/linux/init.h)

**函数解析**：设备驱动卸载函数

`__exitcall_##fn`：定义一个新的 `exitcall_t` 类型的静态变量，并赋值为`fn`

`__exit_call`：`__used __section(".exitcall.exit")`，定义该函数存储的段

&nbsp;

## 4、扩展

> **还记得`__define_initcall`的定义吗？**

```c
#define pure_initcall(fn)       __define_initcall(fn, 0)  
  
#define core_initcall(fn)       __define_initcall(fn, 1)  
#define core_initcall_sync(fn)      __define_initcall(fn, 1s)  
#define postcore_initcall(fn)       __define_initcall(fn, 2)  
#define postcore_initcall_sync(fn)  __define_initcall(fn, 2s)  
#define arch_initcall(fn)       __define_initcall(fn, 3)  
#define arch_initcall_sync(fn)      __define_initcall(fn, 3s)  
#define subsys_initcall(fn)     __define_initcall(fn, 4)  
#define subsys_initcall_sync(fn)    __define_initcall(fn, 4s)  
#define fs_initcall(fn)         __define_initcall(fn, 5)  
#define fs_initcall_sync(fn)        __define_initcall(fn, 5s)  
#define rootfs_initcall(fn)     __define_initcall(fn, rootfs)  
#define device_initcall(fn)     __define_initcall(fn, 6)  
#define device_initcall_sync(fn)    __define_initcall(fn, 6s)  
#define late_initcall(fn)       __define_initcall(fn, 7)  
#define late_initcall_sync(fn)      __define_initcall(fn, 7s)  
  
#define __initcall(fn) device_initcall(fn) 
```

**不同的宏定义，被赋予了不同的调用等级，最后将不同的驱动初始化函数统一汇总到`__initcallx_start`字段统一管理，形成一个有序的列表。**

**这样，我们在内核中，按照顺序遍历这个列表，最后执行对应的模块初始化函数`fn`即可实现驱动的初始化。**

![](https://image-1305421143.cos.ap-nanjing.myqcloud.com/image/image-20231119211155587.png)