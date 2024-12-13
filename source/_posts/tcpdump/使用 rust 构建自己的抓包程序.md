---
title: 使用 Rust 构建自己的抓包程序
categories:
  - tcpdump
tags:
  - tcpdump
  - linux
  - rust
  - net
date: 2024-05-03 17:38:09
---
# Rust Code

```toml
# cargo.toml
[package]
name = "netdump"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
[dependencies]
etherparse = "0.11"
pcap = "0.10"

```



```rust
// main.rs
use etherparse::{IpHeader, PacketHeaders};
use pcap::Device;
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let nic_name = "en0";

    let list: Vec<Device> = pcap::Device::list().unwrap();

    let device = list
        .iter()
        .filter(|d| d.name.eq_ignore_ascii_case(nic_name))
        .last()
        .expect(std::format!("not found nic: {}", nic_name).as_str());

    let mut cap = device.clone().open().unwrap();
    
    while let Ok(packet) = cap.next_packet() {
        match PacketHeaders::from_ethernet_slice(&packet) {
            Err(value) => println!("Err {:?}", value),
            Ok(value) => {
                println!("------------------------------------------------------------------------------");
                match value.link {
                    Some(link_header) => {

                        println!(
                            "link: from {:?} to {:?}",
                            link_header.source, link_header.destination
                        );
                    }
                    None => {}
                }
              
                match value.ip {
                    Some(ip_header) => match ip_header {
                        IpHeader::Version4(ip, _) => {
                            println!("  ipv4: from {:?} to {:?}", ip.source, ip.destination);
                        }
                        IpHeader::Version6(ip, _) => {
                            println!("  ipv6: from {:?} to {:?}", ip.source, ip.destination);
                        }
                    },
                    None => {}
                }
              
                match value.transport {
                    Some(transport_header) => match transport_header.tcp() {
                        Some(tcp) => {
                            println!(
                                "    tcp: from {:?} to {:?}",
                                tcp.source_port, tcp.destination_port
                            );
                        }
                        None => {}
                    },
                    None => {}
                }
            }
        }
    }
    Ok(())
}

```

```rust
cargo build && cargo run

output:
------------------------------------------------------------------------------
link: from [240, 24, 152, 121, 195, 89] to [24, 197, 138, 17, 184, 22]
  ipv4: from [192, 168, 80, 109] to [1, 12, 12, 12]
    tcp: from 62006 to 443
------------------------------------------------------------------------------
...
```

# 为什么可以抓到包?

## TCPDUMP 为什么可以抓到？

```shell
strace tcpdump -i eth0

...
socket(AF_PACKET, SOCK_RAW, 768)
...
bind(3, {sa_family=AF_PACKET, sll_protocol=htons(ETH_P_ALL), sll_ifindex=if_nametoindex("eth0"), sll_hatype=ARPHRD_NETROM, sll_pkttype=PACKET_HOST, sll_halen=0}, 20) = 0
...

```



```c
socket(AF_PACKET, SOCK_RAW, 768)
```

该调用创建了一个 **原始套接字**，允许应用程序通过 **数据链路层（Layer 2）** 访问网络包。它将捕获经过的所有以太网数据包（协议类型为 `ETH_P_ALL`），包括所有不同的协议类型（例如，IP、ARP、IPv6 等）。

- 协议族 (`AF_PACKET`)：通过数据链路层访问网络包。
- 套接字类型 (`SOCK_RAW`)：创建一个原始套接字，允许发送和接收未经处理的原始数据包。
- 协议类型 (`768`)：`ETH_P_ALL`，表示捕获所有以太网帧，不限协议类型。

```c
bind(3, {sa_family=AF_PACKET, sll_protocol=htons(ETH_P_ALL), sll_ifindex=if_nametoindex("eth0"), sll_hatype=ARPHRD_NETROM, sll_pkttype=PACKET_HOST, sll_halen=0}, 20) = 0
```

- 将套接字绑定到 `eth0` 接口，使得该套接字仅接收通过该接口传输的数据包。
- 协议类型设置为 `ETH_P_ALL`，这表示该套接字将接收所有以太网协议的数据包，不限于某个具体协议。
- 数据包类型设置为 `PACKET_HOST`，表示只接收发送到本机的网络数据包。
- 硬件类型设置为 `ARPHRD_NETROM`，用于指定链路层协议类型，尽管在常规的以太网通信中，这个字段通常是 `ARPHRD_ETHER`。

## 分析启动过程

```c
// net/socket.c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
	return __sys_socket(family, type, protocol);
}
```

发起系统调用创建 socket。

```c
// net/socket.c
int __sys_socket(int family, int type, int protocol)
{
	struct socket *sock;
  ...
	sock = __sys_socket_create(family, type, update_socket_protocol(family, type, protocol));
	...
}
```



```c
// net/socket.c
static struct socket *__sys_socket_create(int family, int type, int protocol)
{
	struct socket *sock;
  ...
	retval = sock_create(family, type, protocol, &sock);
  ...
	return sock;
}
```



```c
// net/socket.c
int sock_create(int family, int type, int protocol, struct socket **res)
{
	return __sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
}
```



```c
// net/socket.c
int __sock_create(struct net *net, int family, int type, int protocol,
			 struct socket **res, int kern)
{
	int err;
	struct socket *sock;
	const struct net_proto_family *pf;

	sock = sock_alloc();
    ...
    pf = rcu_dereference(net_families[family]);
	...
	err = pf->create(net, sock, protocol, kern);
	...
}
```

通过 `net_families` 找到 pf, 调用 `create` 方法。

```c
// socket.h
/* Protocol families, same as address families. */
#define PF_UNSPEC	AF_UNSPEC
#define PF_UNIX		AF_UNIX
#define PF_LOCAL	AF_LOCAL
#define PF_INET		AF_INET
#define PF_AX25		AF_AX25
#define PF_IPX		AF_IPX
#define PF_APPLETALK	AF_APPLETALK
#define	PF_NETROM	AF_NETROM
#define PF_BRIDGE	AF_BRIDGE
#define PF_ATMPVC	AF_ATMPVC
#define PF_X25		AF_X25
#define PF_INET6	AF_INET6
#define PF_ROSE		AF_ROSE
#define PF_DECnet	AF_DECnet
#define PF_NETBEUI	AF_NETBEUI
#define PF_SECURITY	AF_SECURITY
#define PF_KEY		AF_KEY
#define PF_NETLINK	AF_NETLINK
#define PF_ROUTE	AF_ROUTE
#define PF_PACKET	AF_PACKET
#define PF_ASH		AF_ASH
#define PF_ECONET	AF_ECONET
#define PF_ATMSVC	AF_ATMSVC
#define PF_RDS		AF_RDS
#define PF_SNA		AF_SNA
#define PF_IRDA		AF_IRDA
#define PF_PPPOX	AF_PPPOX
#define PF_WANPIPE	AF_WANPIPE
#define PF_LLC		AF_LLC
#define PF_IB		AF_IB
#define PF_MPLS		AF_MPLS
#define PF_CAN		AF_CAN
#define PF_TIPC		AF_TIPC
#define PF_BLUETOOTH	AF_BLUETOOTH
#define PF_IUCV		AF_IUCV
#define PF_RXRPC	AF_RXRPC
#define PF_ISDN		AF_ISDN
#define PF_PHONET	AF_PHONET
#define PF_IEEE802154	AF_IEEE802154
#define PF_CAIF		AF_CAIF
#define PF_ALG		AF_ALG
#define PF_NFC		AF_NFC
#define PF_VSOCK	AF_VSOCK
#define PF_KCM		AF_KCM
#define PF_QIPCRTR	AF_QIPCRTR
#define PF_SMC		AF_SMC
#define PF_XDP		AF_XDP
#define PF_MCTP		AF_MCTP
#define PF_MAX		AF_MAX
```

应用层创建时选择的参数是 `AF_PACKET`

`AF_PACKET[address familie]`  => `PF_PACKET[Protocol familie]`

```c
// net/packet/af_packet.c
static const struct net_proto_family packet_family_ops = {
	.family =	PF_PACKET,
	.create =	packet_create,
	.owner	=	THIS_MODULE,
};
```

上述定义的 `create` 方法为 `packet_create`。

```c
// net/packet/af_packet.c
static int packet_create(struct net *net, struct socket *sock, int protocol,
			 int kern)
{
	struct sock *sk;
	struct packet_sock *po;
	__be16 proto = (__force __be16)protocol; /* weird, but documented */
	int err;

	...
	po = pkt_sk(sk);
	// 设置回调函数
    po->prot_hook.func = packet_rcv;
  
    if (proto) {
        po->prot_hook.type = proto;
        __register_prot_hook(sk);
    }

}
```

1. 设置回调函数
2. 注册处理函数

```c
// net/packet/af_packet.c
static void __register_prot_hook(struct sock *sk)
{
	struct packet_sock *po = pkt_sk(sk);
	dev_add_pack(&po->prot_hook);
}
```



```c
// net/core/dev.c
void dev_add_pack(struct packet_type *pt)
{
	struct list_head *head = ptype_head(pt);

	spin_lock(&ptype_lock);
	list_add_rcu(&pt->list, head);
	spin_unlock(&ptype_lock);
}
```



```c
// net/core/dev.c
static inline struct list_head *ptype_head(const struct packet_type *pt)
{
	if (pt->type == htons(ETH_P_ALL))
		return pt->dev ? &pt->dev->ptype_all : &net_hotdata.ptype_all;
}
```

总结：应用层创建一个特殊的 `socket`，设置回调为函数为 `packet_rcv`，并注册到 `ptype_all` 上。

## 分析收包过程

`cpu` 软中断入口

```c
// net/core/dev.c
/**
 *	netif_receive_skb_core - special purpose version of netif_receive_skb
 *	@skb: buffer to process
 *
 *	More direct receive version of netif_receive_skb().  It should
 *	only be used by callers that have a need to skip RPS and Generic XDP.
 *	Caller must also take care of handling if ``(page_is_)pfmemalloc``.
 *
 *	This function may only be called from softirq context and interrupts
 *	should be enabled.
 *
 *	Return values (usually ignored):
 *	NET_RX_SUCCESS: no congestion
 *	NET_RX_DROP: packet was dropped
 */
int netif_receive_skb_core(struct sk_buff *skb)
{
	int ret;

	rcu_read_lock();
	ret = __netif_receive_skb_one_core(skb, false);
	rcu_read_unlock();

	return ret;
}
```



```c
// net/core/dev.c
static int __netif_receive_skb_one_core(struct sk_buff *skb, bool pfmemalloc)
{
	struct net_device *orig_dev = skb->dev;
	struct packet_type *pt_prev = NULL;
	int ret;

	ret = __netif_receive_skb_core(&skb, pfmemalloc, &pt_prev);
	if (pt_prev)
		ret = INDIRECT_CALL_INET(pt_prev->func, ipv6_rcv, ip_rcv, skb,
					 skb->dev, pt_prev, orig_dev);
	return ret;
}
```



```c
// net/core/dev.c
static int __netif_receive_skb_core(struct sk_buff **pskb, bool pfmemalloc,
				    struct packet_type **ppt_prev)
{
	struct packet_type *ptype, *pt_prev;
	rx_handler_func_t *rx_handler;
	struct sk_buff *skb = *pskb;
	struct net_device *orig_dev;
	bool deliver_exact = false;
	int ret = NET_RX_DROP;
	__be16 type;

	net_timestamp_check(!READ_ONCE(net_hotdata.tstamp_prequeue), skb);

	trace_netif_receive_skb(skb);

	list_for_each_entry_rcu(ptype, &skb->dev->ptype_all, list) {
		if (pt_prev)
			ret = deliver_skb(skb, pt_prev, orig_dev);
		pt_prev = ptype;
	}

}
```



```c
// net/core/dev.c
// 触发回调 packet_rcv
static inline int deliver_skb(struct sk_buff *skb,
			      struct packet_type *pt_prev,
			      struct net_device *orig_dev)
{
	if (unlikely(skb_orphan_frags_rx(skb, GFP_ATOMIC)))
		return -ENOMEM;
	refcount_inc(&skb->users);
	return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
}
```



```c
// net/packet/af_packet.c
static int packet_rcv(struct sk_buff *skb, struct net_device *dev,
		      struct packet_type *pt, struct net_device *orig_dev)
{
	enum skb_drop_reason drop_reason = SKB_CONSUMED;
	struct sock *sk = NULL;
	struct sockaddr_ll *sll;
	struct packet_sock *po;
	u8 *skb_head = skb->data;
	int skb_len = skb->len;
	unsigned int snaplen, res;

	if (skb->pkt_type == PACKET_LOOPBACK)
		goto drop;

	sk = pt->af_packet_priv;
	po = pkt_sk(sk);

	if (!net_eq(dev_net(dev), sock_net(sk)))
		goto drop;

	skb->dev = dev;
	...
	res = run_filter(skb, sk, snaplen);
	...
	/* drop conntrack reference */
	nf_reset_ct(skb);
	...
    // 放到 socket 接收队列
	__skb_queue_tail(&sk->sk_receive_queue, skb);
	...
}
```

## 分析发包过程

数据包发往网络设备入口

```c
**
 * __dev_queue_xmit() - transmit a buffer
 * @skb:	buffer to transmit
 * @sb_dev:	suboordinate device used for L2 forwarding offload
 *
 * Queue a buffer for transmission to a network device. The caller must
 * have set the device and priority and built the buffer before calling
 * this function. The function can be called from an interrupt.
 *
 * When calling this method, interrupts MUST be enabled. This is because
 * the BH enable code must have IRQs enabled so that it will not deadlock.
 *
 * Regardless of the return value, the skb is consumed, so it is currently
 * difficult to retry a send to this method. (You can bump the ref count
 * before sending to hold a reference for retry if you are careful.)
 *
 * Return:
 * * 0				- buffer successfully transmitted
 * * positive qdisc return code	- NET_XMIT_DROP etc.
 * * negative errno		- other errors
 */
int __dev_queue_xmit(struct sk_buff *skb, struct net_device *sb_dev)
{
	struct net_device *dev = skb->dev;
	struct netdev_queue *txq = NULL;
	struct Qdisc *q;
	int rc = -ENOMEM;
	bool again = false;


	/* The device has no queue. Common case for software devices:
	 * loopback, all the sorts of tunnels...

	 * Really, it is unlikely that netif_tx_lock protection is necessary
	 * here.  (f.e. loopback and IP tunnels are clean ignoring statistics
	 * counters.)
	 * However, it is possible, that they rely on protection
	 * made by us here.

	 * Check this and shot the lock. It is not prone from deadlocks.
	 *Either shot noqueue qdisc, it is even simpler 8)
	 */
	if (dev->flags & IFF_UP) {
		int cpu = smp_processor_id(); /* ok because BHs are off */

		/* Other cpus might concurrently change txq->xmit_lock_owner
		 * to -1 or to their cpu id, but not to our id.
		 */
		if (READ_ONCE(txq->xmit_lock_owner) != cpu) {
			if (dev_xmit_recursion())
				goto recursion_alert;

			skb = validate_xmit_skb(skb, dev, &again);
			if (!skb)
				goto out;

			HARD_TX_LOCK(dev, txq, cpu);

			if (!netif_xmit_stopped(txq)) {
				dev_xmit_recursion_inc();
				skb = dev_hard_start_xmit(skb, dev, txq, &rc);
				dev_xmit_recursion_dec();
				if (dev_xmit_complete(rc)) {
					HARD_TX_UNLOCK(dev, txq);
					goto out;
				}
			}
    }
  }
}
```



```c
struct sk_buff *dev_hard_start_xmit(struct sk_buff *first, struct net_device *dev,
				    struct netdev_queue *txq, int *ret)
{
	struct sk_buff *skb = first;
	int rc = NETDEV_TX_OK;

	while (skb) {
		struct sk_buff *next = skb->next;
		skb_mark_not_on_list(skb);
		rc = xmit_one(skb, dev, txq, next != NULL);
	}
}
```



```c
static int xmit_one(struct sk_buff *skb, struct net_device *dev,
		    struct netdev_queue *txq, bool more)
{
	unsigned int len;
	int rc;

	if (dev_nit_active(dev))
		dev_queue_xmit_nit(skb, dev);
	return rc;
}
```



```c
void dev_queue_xmit_nit(struct sk_buff *skb, struct net_device *dev)
{
	struct list_head *ptype_list = &net_hotdata.ptype_all;
	struct packet_type *ptype, *pt_prev = NULL;
	struct sk_buff *skb2 = NULL;
  
	if (pt_prev) {
		deliver_skb(skb2, pt_prev, skb->dev);
		pt_prev = ptype;
		continue;
	}
}
```



```c
// net/core/dev.c
// 触发回调 packet_rcv
static inline int deliver_skb(struct sk_buff *skb,
			      struct packet_type *pt_prev,
			      struct net_device *orig_dev)
{
	if (unlikely(skb_orphan_frags_rx(skb, GFP_ATOMIC)))
		return -ENOMEM;
	refcount_inc(&skb->users);
	return pt_prev->func(skb, skb->dev, pt_prev, orig_dev);
}
```



# 参考资料

- https://docs.rs/pcap/latest/pcap/
- https://googleprojectzero.blogspot.com/2017/05/exploiting-linux-kernel-via-packet.html
- https://www.eet-china.com/mp/a78228.html
- https://medium.com/@chnhaoran/tcpdump-deep-dive-how-tcpdump-works-in-the-kernel-b1976ea39f7e

