# Char device demo
## Basic Description
There are three kinds of device drivers, **character device driver**, **block device driver**, **net device driver**.
Character device driver is for some character device, such as led, Button, I2C and so on.

The main work for charater device driver is **register device**, provide the interface for upper application, such as **open**,**release**,**read**,**write**,**ioctl** and so on.

## One Basis Demo
### File: char_dev_test.c
```
    #include <linux/module.h>
    #include <linux/init.h>
    #include <linux/fs.h>
    #include <linux/cdev.h>
    #include <asm/uaccess.h>
    #include <linux/slab.h>
    #include <linux/device.h>

    MODULE_LICENSE("GPL");

    int mychardev_open(struct inode *, struct file *);
    int mychardev_release(struct inode *, struct file *);
    ssize_t mychardev_read(struct file *, char *, size_t, loff_t *);
    ssize_t mychardev_write(struct file *, const char *, size_t, loff_t *);
    ssize_t mychardev_ioctl(struct file *, unsigned int cmd, unsigned long arg);
    int dev_major = 153;
    int dev_minor = 0;

    static struct class *firstdrv_class;
    static struct device *firstdrv_class_dev;

    struct cdev *mychardev_cdev;
    int *gp_testdata;

    struct file_operations mychardev_fops=
    {
        .owner = THIS_MODULE,
        .open = mychardev_open,
        .release = mychardev_release,
        .read = mychardev_read,
        .write = mychardev_write,
        .compat_ioctl = mychardev_ioctl,
        .unlocked_ioctl = mychardev_ioctl,

    };

    static void __exit mychardev_exit(void)
    {
        dev_t devno=MKDEV(dev_major, dev_minor);

        cdev_del(mychardev_cdev);                  // Ask the system to delete the management block
        kfree(mychardev_cdev);
        kfree(gp_testdata);
        unregister_chrdev_region(devno, 1);         // Unregister the devno

        device_unregister(firstdrv_class_dev); // This part is for /dev/node.
        class_destroy(firstdrv_class);

        printk("mychardev unregister success\n");
    }

    static int __init mychardev_init(void)
    {
        int ret, err;
        dev_t devno;
    #if 0
        ret=alloc_chrdev_region(&devno, dev_minor, 1, "mychardev"); // This method can get one MAJOR DEV number dynamically.
        dev_major=MAJOR(devno);

    #else
        devno=MKDEV(dev_major, dev_minor);                        // Static MAJOR DEV number is set, the developer needs to make sure the dev_major is unique.
        ret=register_chrdev_region(devno, 1, "mychardev");
    #endif

        if(ret<0)
        {
            printk("mychardev register failure\n");
            return ret;
        }
        else
        {
            printk("mychardev register success\n");
        }    

        gp_testdata = kmalloc(sizeof(int), GFP_KERNEL);
        memset(gp_testdata,0, sizeof(int));
        mychardev_cdev = kmalloc(sizeof(struct cdev), GFP_KERNEL);
        cdev_init(mychardev_cdev, &mychardev_fops);                        // Add the management block

        mychardev_cdev->owner = THIS_MODULE;

        err=cdev_add(mychardev_cdev, devno, 1);
        if(err<0)
            printk("add device failure\n");

        firstdrv_class = class_create(THIS_MODULE, "mychardev");            // This part add one node at /dev/ to support one interface for application layer.
        // ALL we can omit this procedure and call mknod command to create the dev interface.
        firstdrv_class_dev = device_create(firstdrv_class, NULL, MKDEV(dev_major, 0), NULL,"mychardev");

        printk("register mychardev dev OK\n");

        return 0;
    }
    int mychardev_open(struct inode *inode, struct file *filp)
    {
        filp->private_data = gp_testdata;
        printk("open mychardev dev OK\n");
        return 0;
    }
    int mychardev_release(struct inode *inode, struct file *filp)
    {
        printk("close mychardev dev OK\n");
        return 0;
    }
    ssize_t mychardev_read(struct file *filp, char *buf, size_t len, loff_t *off)
    {
        unsigned int *p_testdata = filp->private_data;
        if(copy_to_user(buf, p_testdata, sizeof(int)))
        {
            return -EFAULT;
        }
        printk("read mychardev dev OK\n");
        return sizeof(int);
    }
    ssize_t mychardev_write(struct file *filp, const char *buf, size_t len, loff_t *off)
    {
        unsigned int *p_testdata = filp->private_data;
        if(copy_from_user(p_testdata, buf, sizeof(int)))
        {
            return -EFAULT;
        }
        printk("write mychardev dev OK\n");
        return sizeof(int);
    }

    ssize_t mychardev_ioctl(struct file * filp, unsigned int cmd, unsigned long arg)
    {
        switch(cmd)
        {
            case 0:
                printk("ioctl 0\n");
                break;
            case 1:
                printk("ioctl 1\n");
                break;
            default:
                printk("ioctl 2\n");

        }
    }
    module_init(mychardev_init);
    module_exit(mychardev_exit);


```
### File: Makefile
```


    obj-m :=char_dev_test.o
    KERNEL := /lib/modules/`uname -r`/build

    all:
    make -C $(KERNEL) M=`pwd` modules

    install:
    make -C $(KERNEL) M=`pwd` modules_install

```

### File: Upper application
```


    #include <sys/types.h>
    #include <sys/stat.h>
    #include <stdio.h>
    #include <fcntl.h>

    int main()
    {
        int fd, num;
        fd=open("/dev/mychardev", O_RDWR);
        if(fd!=-1)
        {
            read(fd, &num, sizeof(int));
            printf("The mychardev is %d\n", num);

            printf("Please input the num written to mychardev\n");
            scanf("%d", &num);
            write(fd, &num, sizeof(int));

            read(fd, &num, sizeof(int));
            printf("The mychardev is %d\n", num);
            
            ioctl(fd, 0 , 0);    //ioctl test
            ioctl(fd, 1, 0 );
            close(fd);
        }
        else
        {
            printf("Device open failure\n");
            perror("open mychardev");
        }
        return 1;
    }


```