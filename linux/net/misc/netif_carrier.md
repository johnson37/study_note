# netif_carrier_on/off/ok

## Introduction
- netif_carrier_on: 网络驱动通知内核网络连接完整
- netif_carrirer_off: 网络驱动通知内核网络断开
- netif_carrier_ok: 查询当前内核认为的网络是连接还是断开

## Code

### Part 1
```
        if (p->p.phy->link)
        {    
            if(!netif_carrier_ok(p->dev))
            {    
                netif_carrier_on(p->dev);
            }    
        }    
        else 
        {    
            if(netif_carrier_ok(p->dev))
            {    
                netif_carrier_off(p->dev);
            }    
        }    
```

### Part 2

```
netif_carrier_on: 
-->linkwatch_fire_event
	-->linkwatch_urgent_event
	-->linkwatch_add_event
	-->linkwatch_schedule_work
		-->queue_delayed_work(lw_wq, &linkwatch_work, delay);
			-->linkwatch_work
			-->linkwatch_event
			-->__linkwatch_run_queue
			-->linkwatch_do_dev
				--> dev_deactivate
				--> dev_activate
				--> netdev_state_change
					-->call_netdevice_notifiers_info

call_netdevice_notifiers_info
-->netdev_notifier_info_init
-->raw_notifier_call_chain
	-->__raw_notifier_call_chain
		-->__raw_notifier_call_chain
			-->notifier_call_chain
				-->nb->notifier_call(nb, val, v)
					-->br_device_event
						-->case NETDEV_UP: br_stp_enable_port
							-->br_init_port [DISABLED --> BLOCKING]
							-->br_port_state_selection(p->br);     [BLOCKING --> FORWARDING ]

#define BR_STATE_DISABLED 0
#define BR_STATE_LISTENING 1
#define BR_STATE_LEARNING 2
#define BR_STATE_FORWARDING 3
#define BR_STATE_BLOCKING 4

add_del_if
	-->br_dev_notify_if_change
		-->raw_notifier_call_chain (BREVT_IF_CHANGED)
			-->notifier_call_chain
				-->nb->notifier_call(nb, val, v)
					-->bridge_notifier [BREVT_IF_CHANGED]
-->bridge_update_pbvlan
```