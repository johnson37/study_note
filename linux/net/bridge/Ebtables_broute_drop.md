# Ebtables broute表和其他表对于DROP的处理区别
## Ebtables broute 的部分实现
**netif_receive_skb_core**
```
netif_receive_skb_core
{
	rx_handler = rcu_dereference(skb->dev->rx_handler);
	if (rx_handler) {
		if (pt_prev) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		switch (rx_handler(&skb)) {
		case RX_HANDLER_CONSUMED:
			ret = NET_RX_SUCCESS;
			goto out;
		case RX_HANDLER_ANOTHER:
			goto another_round;
		case RX_HANDLER_EXACT:
			deliver_exact = true;
		case RX_HANDLER_PASS:
			break;
		default:
			BUG();
		}
	}
}
```
**br_handle_frame**
```
br_handle_frame
{
...
	switch (p->state) {
	case BR_STATE_FORWARDING:
		rhook = rcu_dereference(br_should_route_hook);
		if (rhook) {
			if ((*rhook)(skb)) {
				*pskb = skb;
				return RX_HANDLER_PASS;
			}
			dest = eth_hdr(skb)->h_dest;
		}
}
```
**ebt_broute**
```
static int ebt_broute(struct sk_buff *skb)
{
	struct nf_hook_state state;
	int ret;

	nf_hook_state_init(&state, NULL, NF_BR_BROUTING, INT_MIN,
			   NFPROTO_BRIDGE, skb->dev, NULL, NULL,
			   dev_net(skb->dev), NULL);

	ret = ebt_do_table(skb, &state, state.net->xt.broute_table);
	if (ret == NF_DROP)
		return 1; /* route it */
	return 0; /* bridge it */
}
```
## Ebtables  其他表的行为 (以NAT表的PREROUTING链为例)
**netif_receive_skb_core**
```
	rx_handler = rcu_dereference(skb->dev->rx_handler);
	if (rx_handler) {
		if (pt_prev) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		switch (rx_handler(&skb)) {
		case RX_HANDLER_CONSUMED:
			ret = NET_RX_SUCCESS;
			goto out;
		case RX_HANDLER_ANOTHER:
			goto another_round;
		case RX_HANDLER_EXACT:
			deliver_exact = true;
		case RX_HANDLER_PASS:
			break;
		default:
			BUG();
		}
	}
```
**br_handle_frame**
```
	case BR_STATE_LEARNING:
		if (ether_addr_equal(p->br->dev->dev_addr, dest))
			skb->pkt_type = PACKET_HOST;

		NF_HOOK(NFPROTO_BRIDGE, NF_BR_PRE_ROUTING,
			dev_net(skb->dev), NULL, skb, skb->dev, NULL,
			br_handle_frame_finish);
		break;
```
**NF_HOOK**
```
static inline int
NF_HOOK(uint8_t pf, unsigned int hook, struct net *net, struct sock *sk, struct sk_buff *skb,
	struct net_device *in, struct net_device *out,
	int (*okfn)(struct net *, struct sock *, struct sk_buff *))
{
	return NF_HOOK_THRESH(pf, hook, net, sk, skb, in, out, okfn, INT_MIN);
}
```
**NF_HOOK_THRESH**
```
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
**nf_hook_thresh**
```
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

## Difference between broute and nat/filter
When Drop action is triggered in Broute table, the ret value is **RX_HANDLER_PASS**
When Drop action is triggered in other table, the ret value is **RX_HANDLER_CONSUMED**
```
netif_receive_skb_core
{
	if (rx_handler) {
		if (pt_prev) {
			ret = deliver_skb(skb, pt_prev, orig_dev);
			pt_prev = NULL;
		}
		switch (rx_handler(&skb)) {
		case RX_HANDLER_CONSUMED:
			ret = NET_RX_SUCCESS;
			goto out;
		case RX_HANDLER_ANOTHER:
			goto another_round;
		case RX_HANDLER_EXACT:
			deliver_exact = true;
		case RX_HANDLER_PASS:
			break;
		default:
			BUG();
		}
	}
  ...
  	if (pt_prev) {
		if (unlikely(skb_orphan_frags(skb, GFP_ATOMIC)))
			goto drop;
		else
			ret = pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
	}
  ...
out: 
  return ret;
}
```