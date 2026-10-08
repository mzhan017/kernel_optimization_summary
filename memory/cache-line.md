
# cacheline组
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