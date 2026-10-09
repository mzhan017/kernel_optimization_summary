
# cache
## cacheline组
下面这个是从crash里dump出来的，tcp_sock结构体。
这里__cacheline_group_begin* 和 __cacheline_group_end*，两个不占用内存空间的变量。
这两个变量的作用是什么？
```
struct tcp_sock {
    [0] struct inet_connection_sock inet_conn;
    [1576] __u8 __cacheline_group_begin__tcp_sock_read_tx[0];
    [1576] u32 max_window;
    [1580] u32 rcv_ssthresh;
    [1584] u32 reordering;
    [1588] u32 notsent_lowat;
    [1592] u16 gso_segs;
    [1600] struct sk_buff *lost_skb_hint;
    [1608] struct sk_buff *retransmit_skb_hint;
    [1616] __u8 __cacheline_group_end__tcp_sock_read_tx[0];
    [1616] __u8 __cacheline_group_begin__tcp_sock_read_txrx[0];
    [1616] u32 tsoffset;
    [1620] u32 snd_wnd;
    [1624] u32 mss_cache;
};
```
但是上面这一对（start, end）的作用是做编译器的检查，看变量是否在这个区域，如果被后续的代码改动修改了，就会有编译提示。
比如下面这个net/core/dev.c里的校验
```
static void __init net_dev_struct_check(void)
{
        /* TX read-mostly hotpath */
        CACHELINE_ASSERT_GROUP_MEMBER(struct net_device, net_device_read_tx, priv_flags_fast);
        CACHELINE_ASSERT_GROUP_MEMBER(struct net_device, net_device_read_tx, netdev_ops);
        CACHELINE_ASSERT_GROUP_MEMBER(struct net_device, net_device_read_tx, header_ops);

#ifndef CACHELINE_ASSERT_GROUP_MEMBER
#define CACHELINE_ASSERT_GROUP_MEMBER(TYPE, GROUP, MEMBER) \
        BUILD_BUG_ON(!(offsetof(TYPE, MEMBER) >= \
                       offsetofend(TYPE, __cacheline_group_begin__##GROUP) && \
                       offsetofend(TYPE, MEMBER) <= \
                       offsetof(TYPE, __cacheline_group_end__##GROUP)))
#endif
```
