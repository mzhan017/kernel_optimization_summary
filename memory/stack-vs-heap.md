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
