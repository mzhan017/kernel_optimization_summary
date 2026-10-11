# slack 偷懒

## hrtimer例子
在下面这个函数里使用到了hrtimeout_range里面有一个slack的参数，
作用是这个timeout是否是精确，如果不需要这么精确可以让hrtimer做
性能的优化，减少一些timer唤醒带来的压力，只要在slack的区间都可以timeout，
没有必要完全按照expires参数来决定。

```
schedule_hrtimeout_range，delta = slack
* The @delta argument gives the kernel the freedom to schedule the
 * actual wakeup to a time that is both power and performance friendly
 * for regular (non RT/DL) tasks.
 * The kernel give the normal best effort behavior for "@expires+@delta",
 * but may decide to fire the timer earlier, but no earlier than @expires.

static int poll_schedule_timeout(struct poll_wqueues *pwq, int state,
                                ktime_t *expires, unsigned long slack)
{
    int rc = -EINTR;   // ✅ 初始值直接写死 -EINTR

    set_current_state(state);  // state = TASK_INTERRUPTIBLE

    // 如果还没有IO事件触发，才进入睡眠
    if (!pwq->triggered)
        rc = schedule_hrtimeout_range(expires, slack, HRTIMER_MODE_ABS);

    __set_current_state(TASK_RUNNING); // 唤醒后强制切回运行态

    return rc;
}
```

默认有一个slack值

```
u64 select_estimate_accuracy(struct timespec64 *tv)
{
        u64 ret;
        struct timespec64 now;
        u64 slack = current->timer_slack_ns;

        if (slack == 0)
                return 0;

        ktime_get_ts64(&now);
        now = timespec64_sub(*tv, now);
        ret = __estimate_accuracy(&now);
        if (ret < slack)
                return slack;
        return ret;
}

struct task_struct init_task __aligned(L1_CACHE_BYTES) = {
.timer_slack_ns = 50000, /* 50 usec default slack */

```

Documentation/filesystems/proc.rst
```
3.10    /proc/<pid>/timerslack_ns - Task timerslack value
---------------------------------------------------------
This file provides the value of the task's timerslack value in nanoseconds.
This value specifies an amount of time that normal timers may be deferred
in order to coalesce timers and avoid unnecessary wakeups.

This allows a task's interactivity vs power consumption tradeoff to be
adjusted.

Writing 0 to the file will set the task's timerslack to the default value.

Valid values are from 0 - ULLONG_MAX

An application setting the value must have PTRACE_MODE_ATTACH_FSCREDS level
permissions on the task specified to change its timerslack_ns value.
```