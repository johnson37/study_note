# Netfilter Introduction
## Netfilter brief introduction
Linux 在其网络实现部分，埋了一些钩子点，Netfilter是一个接近于上层的方法，我们通过Netfilter可以将我们的部分代码挂载到对应的钩子点。通过这种方式，我们可以去修改skb中的内容，或者实现对Packet的跟踪（一个调试网络的不错的手段）。

Netfilter并不是一个现成的工具，需要通过代码实现的方式加载自己的钩子函数，具体是进行对包的改造还是对包的跟踪，取决于设计的钩子函数。最终是以模块的形式加载到内核当中。
## Application Sample
```c
    #include <linux/module.h>
    #include <linux/kernel.h>
    #include <linux/init.h>
    #include <linux/types.h>
    #include <linux/netdevice.h>
    #include <linux/skbuff.h>
    #include <linux/netfilter_ipv4.h>
    #include <linux/inet.h>
    #include <linux/in.h>
    #include <linux/ip.h>
          
    MODULE_LICENSE("GPL");
        #define NIPQUAD(addr)\
          ((unsigned char *)&addr)[0],\
          ((unsigned char *)&addr)[1],\
          ((unsigned char *)&addr)[2],\
          ((unsigned char *)&addr)[3]
          
        static unsigned int sample(
        unsigned int hooknum,
        struct sk_buff * skb,
        const struct net_device *in,
        const struct net_device *out,
        int (*okfn) (struct sk_buff *))
    {
        __be32 sip,dip;
         if(skb){
           struct sk_buff *sb = NULL;
           sb = skb;
           struct iphdr *iph;
           iph = ip_hdr(sb);
           sip = iph->saddr;
           dip = iph->daddr;
           printk("Packet for source address: %d.%d.%d.%d\n destination address: %d.%d.%d.%d\n ", NIPQUAD(sip), NIPQUAD(dip));
            }
         return NF_ACCEPT;
    }
          
    struct nf_hook_ops sample_ops = {
           .list = {NULL,NULL},
           .hook = sample,
           .pf = PF_INET,
           .hooknum = NF_INET_PRE_ROUTING,
           .priority = NF_IP_PRI_FILTER+2
        };
          
    static int __init sample_init(void) {
          nf_register_hook(&sample_ops);
          return 0;
        }
          
          
    static void __exit sample_exit(void) {
        nf_unregister_hook(&sample_ops);
        }
         
    module_init(sample_init);
    module_exit(sample_exit);
    MODULE_AUTHOR("johnson.zhao");
    MODULE_DESCRIPTION("Netfilter_trace");

```
**Makefile**
```


    obj-m :=netfilter_test.o
    KERNEL := /lib/modules/`uname -r`/build
    all:
    make -C $(KERNEL) M=`pwd` modules
    install:
    make -C $(KERNEL) M=`pwd` modules_install


```

这里只是在一个钩子点加上了一个回调函数，理论上Linux网络中埋的钩子点都是可以加钩子的，并且每一个钩子点是可以挂载多个回调函数的，多个回调函数之间的调用先后关系，取决于其中的优先级选项。

## Mechanism
Linux中的钩子点是用一个二维数组进行维护，第一个维度是不同的协议簇，比如说ipv4,ipv6,arp,bridge 等等，第二个维度是具体每一个协议族中执行的关键节点。
```c
struct list_head nf_hooks[NFPROTO_NUMPROTO][NF_MAX_HOOKS];
```
### Register Hook
```c
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *reg)
{
	struct list_head *hook_list;
	struct nf_hook_entry *entry;
	struct nf_hook_ops *elem;
 ...
   	mutex_lock(&nf_hook_mutex);
	list_for_each_entry(elem, hook_list, list) {
		if (reg->priority < elem->priority)
			break;
	}
	list_add_rcu(&entry->ops.list, elem->list.prev);
	mutex_unlock(&nf_hook_mutex);
```
### Usage HOOK
```c
NF_HOOK(NFPROTO_ARP, NF_ARP_FORWARD, state->sk, skb,
        state->in, state->out, br_nf_forward_finish);
```
```c
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk, struct sk_buff *skb,
	struct net_device *in, struct net_device *out,
	int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	return NF_HOOK_THRESH(pf, hook, net, sk, skb, in, out, okfn, INT_MIN);
}
```
```c
static inline int
NF_HOOK_THRESH(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk,
	       struct sk_buff *skb, struct net_device *in,
	       struct net_device *out,
	       int (*okfn)(struct net *, struct sock *, struct sk_buff *),
	       int thresh)
{
	int ret = nf_hook_thresh(pf, hook, net, sk, skb, in, out, okfn, thresh);
	if (ret == 1)
		ret = okfn(net, sk, skb);
	return ret;
}
```
```c
static inline int nf_hook_thresh(u_int8_t pf, unsigned int hook,
				 struct net *net,
				 struct sock *sk,
				 struct sk_buff *skb,
				 struct net_device *indev,
				 struct net_device *outdev,
				 int (*okfn)(struct net *, struct sock *, struct sk_buff *),
				 int thresh)
{
	struct list_head *hook_list = &net->nf.hooks[pf][hook];

	if (nf_hook_list_active(hook_list, pf, hook)) {
		struct nf_hook_state state;

		nf_hook_state_init(&state, hook_list, hook, thresh,
				   pf, indev, outdev, sk, net, okfn);
		return nf_hook_slow(skb, &state);
	}
	return 1;
}
```
```c
int nf_hook_slow(struct sk_buff *skb, struct nf_hook_state *state)
{
	struct nf_hook_ops *elem;
	unsigned int verdict;
	int ret = 0;

	/* We may already have this, but read-locks nest anyway */
	rcu_read_lock();

	elem = list_entry_rcu(state->hook_list, struct nf_hook_ops, list);
next_hook:
	verdict = nf_iterate(state->hook_list, skb, state, &elem);
	if (verdict == NF_ACCEPT || verdict == NF_STOP) {
		ret = 1;
	} else if ((verdict & NF_VERDICT_MASK) == NF_DROP) {
		kfree_skb(skb);
		ret = NF_DROP_GETERR(verdict);
		if (ret == 0)
			ret = -EPERM;
	} else if ((verdict & NF_VERDICT_MASK) == NF_QUEUE) {
		int err = nf_queue(skb, elem, state,
				   verdict >> NF_VERDICT_QBITS);
		if (err < 0) {
			if (err == -ESRCH &&
			   (verdict & NF_VERDICT_FLAG_QUEUE_BYPASS))
				goto next_hook;
			kfree_skb(skb);
		}
	}
	rcu_read_unlock();
	return ret;
}
```
```c
unsigned int nf_iterate(struct list_head *head,
			struct sk_buff *skb,
			struct nf_hook_state *state,
			struct nf_hook_ops **elemp)
{
	unsigned int verdict;

	/*
	 * The caller must not block between calls to this
	 * function because of risk of continuing from deleted element.
	 */
	list_for_each_entry_continue_rcu((*elemp), head, list) {
		if (state->thresh > (*elemp)->priority)
			continue;

		/* Optimization: we don't need to hold module
		   reference here, since function can't sleep. --RR */
repeat:
		verdict = (*elemp)->hook((*elemp)->priv, skb, state);
		if (verdict != NF_ACCEPT) {
#ifdef CONFIG_NETFILTER_DEBUG
			if (unlikely((verdict & NF_VERDICT_MASK)
							> NF_MAX_VERDICT)) {
				NFDEBUG("Evil return from %p(%u).\n",
					(*elemp)->hook, state->hook);
				continue;
			}
#endif
			if (verdict != NF_REPEAT)
				return verdict;
			goto repeat;
		}
	}
	return NF_ACCEPT;
}
```