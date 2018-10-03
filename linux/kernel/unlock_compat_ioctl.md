#  unlock_ioctl 与 compat_ioctl
```c

	struct file_operations led_fops = {
	.owner          = THIS_MODULE,
	.open           = led_open,
	.release        = led_release,
	.compat_ioctl   = led_ioctl,
	.unlocked_ioctl = led_ioctl,
	};
```

##  unlock_ioctl
unlock_ioctl 是原来ioctl的升级。相对于以前的.ioctl , 现在的.unlock_ioctl 是为了撤销大粒度锁设计的。

### 大内核锁 ###
何为大内核锁？（BKL）
大内核锁(BKL)的设计是在kernel hacker们对多处理器的同步还没有十足把握时，引入的大粒度锁。他的设计思想是，一旦某个内核路径获取了这把锁，那么其他所有的内核路径都不能再获取到这把锁。自旋锁加锁的对象一般是一个全局变量，大内核锁加锁的对象是一段代码，里面可能包含多个全局变量。那么他带来的问题是，虽然A只需要互斥访问全局变量a，但附带锁了全局变量b，从而导致B不能访问b了。 

大内核锁最先的实现靠一个全局自旋锁，但大家觉得这个锁的开销太大了，影响了实时性，因此后来将自旋锁 改成了mutex，但阻塞时间一般不是很长，所以加锁失败的挂起和唤醒也是非常costly 所以后来又改成了自旋锁实现。大内核锁一般是在文件系统，驱动等中用的比较多。目前kernel hacker们仍然在努力将大内核锁从linux里铲除。 

## compat_ioctl 
compat_ioctl 是为了适应32位app在64位kernel上运行的适配问题，返回值是32位还是64位的问题。
CPU从32位升级到64位，对应的OS也由32位升级到了64位，但是有些早已经完成的app是在32位系统下编译产生的，如果希望不不重新编译源码的情况下，在新的64位OS上使用，新的64位OS势必需要对32位app在64位kernel上运行的支持。
