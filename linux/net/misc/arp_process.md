# ARP Process

```
netif_receive_skb_core
-->pt_prev->func(skb, skb->dev, pt_prev, orig_dev);     [For arp , pt_prev->func == arp_rcv]
-->arp_rcv  [arp.c]
-->NF_HOOK(NFPROTO_ARP, NF_ARP_IN, skb, dev, NULL, arp_process)
-->arp_process
-->arp_send
-->arp_create
-->arp_xmit
-->dev_queue_xmit
-->dev_hard_start_xmit
-->ops->ndo_start_xmit(skb, dev)
---------------------------------------------------------------------------------------------
Net Driver Layer

```