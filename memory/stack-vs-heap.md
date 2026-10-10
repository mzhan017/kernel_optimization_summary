# overview
在优化的领域，从堆到栈的转换也占有一席之地
## heap to stack
在select/poll的要监听的fd比较少的情况下，优选stack内存，而不是kmalloc
https://github.com/torvalds/linux/commit/70674f95c0a2ea694d5c39f4e514f538a09be36f
```
[PATCH] Optimize select/poll by putting small data sets on the stack
Optimize select and poll by a using stack space for small fd sets

/* ~832 bytes of stack space used max in sys_select/sys_poll before allocating
   additional memory. */
#define MAX_STACK_ALLOC 832
#define FRONTEND_STACK_ALLOC	256
#define SELECT_STACK_ALLOC	FRONTEND_STACK_ALLOC
#define POLL_STACK_ALLOC	FRONTEND_STACK_ALLOC
#define WQUEUES_STACK_ALLOC	(MAX_STACK_ALLOC - FRONTEND_STACK_ALLOC)
#define N_INLINE_POLL_ENTRIES	(WQUEUES_STACK_ALLOC / sizeof(struct poll_table_entry))
```

在实际的改动中，对齐有关系（而对齐也是优化的一个领域），比如下面这个代码，下面的代码有说
需要long型的数组，这样可以更好的适合64位机上的对齐。
```
static int do_sys_poll(struct pollfd __user *ufds, unsigned int nfds,
		struct timespec64 *end_time)
{
	struct poll_wqueues table;
	int err = -EFAULT, fdcount;
	/* Allocate small arguments on the stack to save memory and be
	   faster - use long to make sure the buffer is aligned properly
	   on 64 bit archs to avoid unaligned access */
	long stack_pps[POLL_STACK_ALLOC/sizeof(long)];
	struct poll_list *const head = (struct poll_list *)stack_pps;
 	struct poll_list *walk = head;
```
