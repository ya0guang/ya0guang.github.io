---
layout: single
title: Kernel Module 历险记
date: 2021-01-17 14:10:07
categories: "Notes"
tags:
- Notes
- Linux Kernel
header:
  teaser: /images/Linux/chuckvstux.jpg
---

最近在我大哥[@StanPlatinum](https://drstanliu.cn/)的指导下搞了个截获/添加syscall的kernel module，在这里稍微记录一下折腾的过程。

前排特别感谢一下**Robotxm**大佬的[这篇博文](https://moefactory.com/3041.moe)里面记录了他详细的折腾过程。大佬本科期间就开始吊着Linux Kernel到处捉迷藏了，我本科的时候还不知道在哪个田野里面玩泥巴呢。

# Kernel Module

简单地说就是给Kernel里面塞点奇奇怪怪的东西让Kernel变得奇怪起来。框架大抵是这样的：

```c
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>

MODULE_LICENSE("GPL");

// user whatever function name as you wish
static int __init init_module(void) {
    //your code
}

// user whatever function name as you wish
static void __exit exit_module(void) {
    //your code
}

module_init(init_module);
module_exit(exit_module);
```

用特殊的姿势编译就好了：
```Makefile
obj-m += [source_file_name].o

all:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules

clean:
	make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
```

我的系统环境是Ubuntu 20.04，Kernel是5.4.x。

# `syscall` Hook

我的目的是Hook一个syscall。具体的一些hook的方法可以参考[这篇文章](https://medium.com/bugbountywriteup/linux-kernel-module-rootkit-syscall-table-hijacking-8f1bc0bd099c)。但是他给出的代码只有一个大概的framework，大多数都没有在我的kernel下奏效。但是基本思路是完全可以借鉴的。

Hook一个system call大概需要如下几个步骤：
1. 找到syscall table的位置
2. 找到你想要hook的syscall handler function在syscall table中的位置
3. 修改权限使得syscall table变为可写
4. 记录下来原本的处理函数指针（如果你不希望影响正常的系统功能的话）
5. 把这个位置指向的函数改成你自己的
6. 还原syscall table的权限

```C
#include <linux/module.h>
#include <linux/kernel.h>
#include <linux/init.h>
#include <linux/unistd.h>
#include <linux/time.h>
#include <linux/uaccess.h>
#include <linux/sched.h>
#include <linux/kallsyms.h>

#define __NR_passpid 336

MODULE_LICENSE("GPL");
char *sym_name = "sys_call_table";

typedef asmlinkage long (*sys_call_ptr_t)(const struct pt_regs *);
static sys_call_ptr_t *sys_call_table;
unsigned int stored_cr0;

sys_call_ptr_t ori_futex;
sys_call_ptr_t ori_passpid;
long tracked_pid;

// borrowed from Robotxm
unsigned int clear_and_return_cr0(void)
{
    unsigned int cr0 = 0;
    unsigned int ret;
    // 64 位系统，借助 RAX 寄存器
    asm volatile("movq %%cr0, %%rax"
                 : "=a"(cr0)); // 将 CR0 的值移动到 RAX 并输出到变量 cr0 中
    ret = cr0;
    cr0 &= 0xfffeffff;                            // 将变量 cr0 值的第 16 位置零，并将修改后的值写入 CR0 寄存器
    asm volatile("movq %%rax, %%cr0" ::"a"(cr0)); // 将 CR0 中的值读到 RAX 中，再将 RAX 的值放到 EAX 中
    return ret;
}

void setback_cr0(unsigned int val)
{
    asm volatile("movq %%rax, %%cr0" ::"a"(val));
}

static asmlinkage long my_futex(const struct pt_regs *regs)
{
    if (current->pid == tracked_pid && tracked_pid != 0)
    {
        int futex_op = (int)regs->si;
        uint32_t val = (uint32_t)regs->dx;
        uint32_t *uaddr = (uint32_t *)regs->di;
        struct timespec timeout;
        memset(&timeout, 0, sizeof(struct timespec));
        printk("[%d], *uaddr: 0X%p, futex_op: %d, val: %u\n", current->pid, uaddr, futex_op, val);
        if (regs->r10 != NULL)
        {
            copy_from_user(&timeout, regs->r10, sizeof(struct timespec));
            pr_info("[%d] sec: %ld, nsec: %ld\n", current->pid, timeout.tv_sec, timeout.tv_nsec);
        }
        put_user(11037, uaddr);
        return 0;
    }
    return ori_futex(regs);
}

static asmlinkage long passpid(const struct pt_regs *regs)
{
    long result = 0;
    tracked_pid = (long long)regs->di;
    printk("passpid: got pid [%ld]\n", tracked_pid);
    return result;
}

static int __init hello_init(void)
{
    sys_call_table = (sys_call_ptr_t *)kallsyms_lookup_name(sym_name);
    ori_futex = sys_call_table[__NR_futex];
    ori_passpid = sys_call_table[__NR_passpid];
    tracked_pid = 0;
    // Temporarily disable write protection
    stored_cr0 = clear_and_return_cr0();
    sys_call_table[__NR_futex] = my_futex;
    // insert our own syscall to pass pid to monitor
    sys_call_table[__NR_passpid] = passpid;
    // Re-enable write protection
    setback_cr0(stored_cr0);

    return 0;
}

static void __exit hello_exit(void)
{
    // Temporarily disable write protection
    stored_cr0 = clear_and_return_cr0();
    sys_call_table[__NR_futex] = ori_futex;
    sys_call_table[__NR_passpid] = ori_passpid;
    // Re-enable write protection
    setback_cr0(stored_cr0);
}

module_init(hello_init);
module_exit(hello_exit);
```

这次选择的受害者是`futex`函数，将之套在了`my_futex`里面。我还添加了一个syscall，`passpid`，来传递一个pid使得我能够做到只监控一个pid对于`futex`的所有调用。

# 踩坑

详细的过程直接看代码就可以了，这里不得不提踩过的无数坑。

## 内核参数传递

64位系统Kernal的参数传递使用了六个寄存器，传递顺序是*di*, *si*, *dx*, *cx*, *r8*, *r9*。参数可能是64bit长度也可能是32bit长度所以寄存器的前缀可以为*e*或者*r*。这些参数通过`struct pt_regs`被传递。  
一开始的时候我以为只要用到linux manual(2) syscall的signature就好了，做一个函数指针，结果啥啥读不出来。

## 指针内容获取

原则上讲在开启了[SMAP](https://en.wikipedia.org/wiki/Supervisor_Mode_Access_Prevention)的机器上是没有办法直接从kernel访问user space的memory的。这里通过`copy_from_user`和`put_user`两个函数解决从user space拿到一段memory和置user space一个内存上的值的问题。其他相关函数可以在[这里](https://www.kernel.org/doc/htmldocs/kernel-api/ch04s02.html)看到说明。

## pid获取

感觉这是非常神奇的设定，直接去access `current->pid`就可以拿到pid了！！！不知道这个`current`指针是在哪里被定义的。

## 添加新的syscall

请注意不要把原本的syscall table的正常功能破坏掉！Linux在table中预留了很多空的slot，详见reference。这里需要注意的是syscall如果不存在的话硬尻回返回`-1`。而硬尻一个syscall的方法如下：
```cpp
#include <iostream>
#include <unistd.h>
using namespace std;

#define PASSPID_SYSCALLID 336

int main()
{
    int a;
    cin >> a;
    cout << a << endl;
    cout << syscall(PASSPID_SYSCALLID, a) << endl;
    return 0;
}
```

# 心中的几根刺

- 用来获取syscall table的函数`kallsyms_lookup_name`似乎在5.9之后就么得了，不知道那时应该怎么操作
- 在`rmmod`之后系统有时候会崩。推测是因为有futex正在被处理的时候更改了syscall table导致一些futex无法被正确的处理。具体的错误信息如下：
    ```shell
    [292887.945127] BUG: unable to handle page fault for address: ffffffffc103a06e
    [292887.945127] #PF: supervisor instruction fetch in kernel mode
    [292887.945128] #PF: error_code(0x0010) - not-present page
    ```

# Reference

- [futex(2)](https://man7.org/linux/man-pages/man2/futex.2.html)
- [linux 5.4 syscall_64.tbl](https://github.com/torvalds/linux/blob/v5.4/arch/x86/entry/syscalls/syscall_64.tbl)