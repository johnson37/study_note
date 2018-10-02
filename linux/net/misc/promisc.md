# 网卡模式
## 网卡模式的种类
- 广播模式：处在该模式下的网卡可以接收广播包
- 多播模式：处在该模式下的网卡可以接收组播包
- 单薄模式：处在该模式下的网卡只能接收目的MAC为自身的报文
- 混杂模式：处在该模式下的网卡可以接收通过该网卡的一切数据报文

## 混杂模式介绍
<font color=red>混杂模式就是接收所有经过网卡的数据包，包括不是发给本机的包。默认情况下网卡只把发给本机的包（包括广播包）传递给上层程序，其它的包一律丢弃。简单的讲,混杂模式就是指网卡能接受所有通过它的数据流，不管是什么格式，什么地址的。事实上，计算机收到数据包后，由网络层进行判断，确定是递交上层（传输层），还是丢弃，还是递交下层（数据链路层、MAC子层）转发。</font> 
  通常在需要用到抓包工具，例如ethereal、sniffer、capsa时，需要把网卡置于混杂模式 
## 混杂模式的设置
- **ifconfig eth0 promisc**
- **ifconfig eth0 -promisc**

- [root@mip-123456 ioctl]# ifconfig eth0
- eth0 Link encap:Ethernet HWaddr 00:22:68:3C:9C:F0 
-          inet addr:172.24.149.212 Bcast:172.24.149.255 Mask:255.255.255.0
-           inet6 addr: fe80::222:68ff:fe3c:9cf0/64 Scope:Link
-           UP BROADCAST RUNNING <font color=red>**PROMISC**</font> MULTICAST MTU:1500 Metric:1
-           RX packets:3653 errors:0 dropped:0 overruns:0 frame:0
-           TX packets:6630 errors:0 dropped:0 overruns:0 carrier:0
-           collisions:0 txqueuelen:1000 
-           RX bytes:6255432 (5.9 MiB) TX bytes:2379200 (2.2 MiB)
- 	    Interrupt:233 Base address:0xa000 

