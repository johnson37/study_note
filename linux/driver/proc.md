# Proc File
## Background
- Linux中通常使用proc中的文件来存储一些系统信息。如果需要调试内核模块的时候，可以考虑添加proc文件来辅助调试。
- /proc 文件夹的文件系统是一个虚拟的文件系统，不能够通过普通文件的创建，读写来实现。
- /proc 下文件夹以及文件的创建，读写类似于驱动。

## Usage
### proc_test.c
**Note: 这里的seq_read, seq_release都是系统已经实现的函数，调用默认行为。seq_read 会读取的一个seq file，该seq file会在proc_open中打开，只要在对应的seq file中完成相应的填充，则在上层在读取该文件时（cat /proc/abc_proc），就会读取该seq_file中的字符串。**

```c


    #include <linux/module.h>
    #include <linux/sched.h>
    #include <linux/uaccess.h>
    #include <linux/proc_fs.h>
    #include <linux/fs.h>
    #include <linux/seq_file.h>
    #include <linux/slab.h>

    static char * str = NULL;

    static int my_proc_show(struct seq_file *m, void * v)
    {
        seq_printf(m,"current kernel time is %ld\n", jiffies);
        seq_printf(m,"str is %s\n", str);
        return 0;
    }


    static int my_proc_open(struct inode * inode, struct file *file)
    {
        return single_open(file, my_proc_show, NULL);
    }

    static struct file_operations my_fops =
    {
        .owner = THIS_MODULE,
        .open = my_proc_open,
        .release = single_release,
        .read = seq_read,
        .llseek = seq_lseek,
    };

    static int __init my_init(void)
    {
        struct proc_dri_entry *file;
        file = proc_create("abc_proc", 0644, NULL, &my_fops);
        if (!file)
            return -ENOMEM;
        return 0;    
    }

    static void __exit my_exit(void)
    {
        remove_proc_entry("abc_proc", NULL);
        kfree(str);
    }

    module_init(my_init);
    module_exit(my_exit);
    MODULE_LICENSE("GPL");
    MODULE_AUTHOR("Johnson");


```

### Makefile
```


    obj-m :=proc_test.o
    KERNEL := /lib/modules/`uname -r`/build

    all:
    make -C $(KERNEL) M=`pwd` modules

    install:
    make -C $(KERNEL) M=`pwd` modules_install


```