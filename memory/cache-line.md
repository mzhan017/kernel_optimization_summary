
# cache
## cacheline组
cacheline组的主要目的是提高相关变量的 ‘data locality’，靠的近一点，最好是放到一个cache line里，
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
### 相关commit
cacheline group的引入：
https://github.com/torvalds/linux/commit/aeb9ce058d7c6193dc41e06b3a5b29d22c446b14
```
cache: enforce cache groups
Set up build time warnings to safeguard against future header changes of
organized structs.

Warning includes:
1) whether all variables are still in the same cache group
2) whether all the cache groups have the sum of the members size (in the
   maximum condition, including all members defined in configs)

The __cache_group* variables are ignored in kernel-doc check in the
various header files they appear in to enforce the cache groups.
```
上面的commit虽然加强了group的概念，但是group没有实现cacheline对齐，下面这个commit做了。但是
这个改动太大，可能导致结构体的size变大不小。所以要有所取舍了。
https://github.com/torvalds/linux/commit/2cb13dec8c5e5e104fd2f71c2dee761d6ed9a333
```
cache: add __cacheline_group_{begin, end}_aligned() (+ couple more)
__cacheline_group_begin(), unfortunately, doesn't align the group
anyhow. If it is wanted, then you need to do something like

__cacheline_group_begin(grp) __aligned(ALIGN)

which isn't really convenient nor compact.
Add the _aligned() counterparts to align the groups automatically to
either the specified alignment (optional) or ``SMP_CACHE_BYTES``.
Note that the actual struct layout will then be (on x64 with 64-byte CL):

struct x {
	u32 y;				// offset 0, size 4, padding 56
	__cacheline_group_begin__grp;	// offset 64, size 0
	u32 z;				// offset 64, size 4, padding 4
	__cacheline_group_end__grp;	// offset 72, size 0
	__cacheline_group_pad__grp;	// offset 72, size 0, padding 56
	u32 w;				// offset 128
};

```

关于 CACHELINE_ASSERT_GROUP_SIZE 断言的使用问题
https://github.com/torvalds/linux/commit/e1b022c2bdf1f2a631340b1b2ef265090534f65a
```
tcp: remove CACHELINE_ASSERT_GROUP_SIZE() uses
Maintaining the CACHELINE_ASSERT_GROUP_SIZE() uses
for struct tcp_sock has been painful.

This had little benefit, so remove them.
```

关于tcp_sock结构体的变动
https://github.com/torvalds/linux/commit/3afb106f3f9aa81c512ec5c7e2f7e1c01a2a6e6b
```
tcp: move few fields for data locality

After recent additions (PSP and AccECN) I wanted to make another
round on fields locations to increase data locality.
```
