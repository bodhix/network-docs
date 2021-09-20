### seccomp

### 简介

之前只知道，Docker 使用的核心技术是 namespace 和 cgroup。这几天，发现一个新的技术点：seccomp。seccomp 是 secure computing mode 的缩写，意为构造一个安全的计算环境 -- 限制服务对系统调用（syscall）的使用。

seccomp 主要经历两个阶段：

1. 从内核2.6.23引入的，通过 prctl 设置，设置之后，该进程只支持以下四个基本的系统调用：read / write / _exit / sigreturn。由于限制太死了，并不是很实用。
2. 在内核3.5，使用基于 BPF 的系统调用过滤功能，渐渐强大起来。支持指定 filter 以及对于的 action。不过，实际编写 BPF 还是比较麻烦，所以就有了 [libseccomp](https://github.com/seccomp/libseccomp) 库，使用就比较方便了。

当前的使用，则表现为两种 seccomp 模式：

1. SECCOMP_SET_MODE_STRICT
2. SECCOMP_SET_MODE_FILTER

#### 过滤条件 filter

1. 系统调用：指定特定的系统调用作为过滤条件，匹配上则执行对应的动作

   获取某个系统调用的编号：/usr/include/x86_64-linux-gnu/asm/unistd_64.h

2. 系统调用的参数：可以对于同个系统调用，根据参数不同指定不同的动作。比如只允许 dup 标准输出，不能 dup 标准错误等

#### 执行动作 action

1. SCMP_ACT_KILL
2. SCMP_ACT_KILL_PROCESS：kill 掉整个进程树
3. SCMP_ACT_TRAP：抛出 SIGSYS 信号
4. SCMP_ACT_ERRNO：返回指定错误码
5. SCMP_ACT_TRACE
6. SCMP_ACT_LOG：syslog 打个日志，其他的不变
7. SCMP_ACT_ALLOW：正常调用
8. SCMP_ACT_NOTIFY：触发通知

以上通过 man seccomp_rule_add 可以获取相关信息。

**这里面最有意思的，应该是 ACT_NOTIFY 了，触发通知，可以做很多事情。而且，如果使用 notification，那么调用对应系统调用的进程会被 block 住，直到监控进程有对应的 seccomp_notify_respond。**

### 使用

最主要的使用场景，就是限制 container（包括 Docker 和 LCX）的可以调用的系统调用，或者是对 container 要进行的系统调用进行鉴权 -- container 调用某个系统调用后 block 住，同时 notify 给监控程序。监控程序进行严格的权限校验之后，返回结果，内核再根据监控程序返回结果，处理 container 系统调用的结果。

简单的示例程序：

```c
/* test for linux-feature
   secure computing, filter syscall
 */
#include <stdio.h>          // for printf
#include <sys/prctl.h>      // for prctl
#include <linux/seccomp.h>  // for seccomp constant
#include <unistd.h>         // for dup / access
#include <fcntl.h>          // for open
#include <errno.h>          // for errno
#include <seccomp.h>        // for seccomp_xxx


static int set_seccomp_strict()
{
    printf("setting seccomp strict\n");
    prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT);
}

int test_seccomp_strict()
{
    printf("before set seccomp strict\n");
    dup(1);

    set_seccomp_strict();

    dup(2);
    printf("YOU SHOULD NOT SEE ME!!!\n");
}

static int set_seccomp_filter()
{
    scmp_filter_ctx ctx;
    ctx = seccomp_init(SCMP_ACT_KILL);  // default action: kill

    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(write), 0);
    seccomp_rule_add(ctx, SCMP_ACT_LOG, SCMP_SYS(access), 0);
    seccomp_rule_add(ctx, SCMP_ACT_ERRNO(16), SCMP_SYS(openat), 0);

    // use args as part of filter
    seccomp_rule_add(ctx, SCMP_ACT_ALLOW, SCMP_SYS(dup), 1,
                     SCMP_A0(SCMP_CMP_EQ, 1));

    seccomp_load(ctx);
}

int test_seccomp_filter()
{
    int fd = 0;
    int ret = 0;
    fd = open("/etc/hosts", O_RDONLY);
    printf("open(%d) file /etc/hosts fd: %d\n", SCMP_SYS(openat), fd);

    set_seccomp_filter();

    fd = open("/etc/hosts", O_RDONLY);  // get errno 16 here
    printf("open(%d) file /etc/hosts fd: %d, errno: %d\n", SCMP_SYS(openat), fd, errno);

    ret = access("/etc/hosts", F_OK);
    printf("access(%d) file /etc/hosts ret: %d\n", SCMP_SYS(access), ret);

    (void)dup(1);
    printf("dup(%d) fd 1\n", SCMP_SYS(dup));

    (void)dup(2);
    printf("YOU SHOULD NOT SEE ME!!!\n");
}

int main()
{
    // test_seccomp_strict();
    test_seccomp_filter();
    return 0;
}

```

### 原理与实现

通过简单分析，可以知道 seccomp 机制的实现原理主要有以下几个步骤：

1. 通过系统调用，给当前进程设置上 seccomp 机制（strict 还是 filter），所以当前进程（current）应该有对应的数据结构存储这些信息
2. 当进程进行系统调用的时候，判断进程是否设置了 seccomp，如果是，则根据设置规则进行

#### 设置 seccomp 机制

实际上，也是如此：系统调用是 seccomp，会设置 task 结构中的一个标记(task->seccomp.mode)，如果是 filter，还会将filter设置到 task 中(task->seccomp.filter)。

SYSCALL seccomp --> do_seccomp --> seccomp_set_mode_strict --> seccomp_assign_mode

从代码上看，不允许直接修改内核 seccomp 模式。

```c
// kernel/seccomp.c
static inline void seccomp_assign_mode(struct task_struct *task,
				       unsigned long seccomp_mode)
{
	assert_spin_locked(&task->sighand->siglock);

	task->seccomp.mode = seccomp_mode;
	/*
	 * Make sure TIF_SECCOMP cannot be set before the mode (and
	 * filter) is set.
	 */
	smp_mb__before_atomic();
	set_tsk_thread_flag(task, TIF_SECCOMP);
}
```

#### 触发 seccomp 机制

在系统调用入口，做一层拦截，判断进程是否设置了 task->seccomp.mode，如果是，则根据 mode 对应处理。

```c
__NR_seccomp_read,
__NR_seccomp_write,
__NR_seccomp_exit,
__NR_seccomp_sigreturn
```

对于 filter (__seccomp_filter)，也简单，seccomp_run_filters 获取当前 syscall nr，然后执行 BPF 命令，看是否匹配上，匹配上，则将 action 返回。再 switch(action)，根据不同的 action 做不同的动作

```c
// arch/x86/entry/common.c
do_syscall_64
  |-- syscall_trace_enter
      |-- __secure_computing

    
// kernel/seccomp.c
int __secure_computing(const struct seccomp_data *sd)
{
  int mode = current->seccomp.mode;
  int this_syscall;

  if (IS_ENABLED(CONFIG_CHECKPOINT_RESTORE) &&
      unlikely(current->ptrace & PT_SUSPEND_SECCOMP))
    return 0;

  this_syscall = sd ? sd->nr :
    syscall_get_nr(current, task_pt_regs(current));

  switch (mode) {
  case SECCOMP_MODE_STRICT:
    __secure_computing_strict(this_syscall);  /* may call do_exit */
    return 0;
  case SECCOMP_MODE_FILTER:
    return __seccomp_filter(this_syscall, sd, false);
  default:
    BUG();
  }
}
```

### Seccomp Notification

#### 功能

使用 seccomp filter 模式的时候，可以使用 user-notify 功能，这就是 seccomp notification 功能。其基本作用是，进程设置该功能之后，当进程调用对应的系统调用时，会触发通知，同时该进程会 block 住。当监控通知的进程进行业务处理，然后返回给内核的时候（respond），block 的进程才会继续走下去。利用该功能，我们可以修改系统调用的行为（具体行为、返回内容、返回值等）

#### 原理

seccomp notification 的机制比较奇怪，大致原理如下：将进程设置使用 seccomp notification 的时候，该进程会在内核创建一个 anon_inode。进程可以通过系统调用获取该 anon_inode 的 fd。当触发 notification 的时候，该 fd 可读，用户态的处理结果，也通过该 fd 返回给内核。

这就导致一个问题，要么监控程序和业务程序（设置 seccomp 的进程）必须是父子进程，要么将监控 fd 传递给其他进程，这就是跨进程传递文件描述符。使用 Unix Socket 的 sendmsg / recvmsg，其参数带有控制信息，可以跨进程传递文件描述符（备注：send/sendto 只能发送 buffer，没有携带控制信息，无法传递文件描述符）

#### 示例

```c
/* test for linux-feature
   secure computing, filter syscall
 */
#include <stdio.h>          // for printf
#include <stdlib.h>         // for exit
#include <sys/wait.h>       // for waitpid
#include <linux/seccomp.h>  // for seccomp constant
#include <unistd.h>         // for dup / fork / access
#include <sys/types.h>      // for pid_t
#include <seccomp.h>        // for seccomp_xxx


static int set_seccomp_notify()
{
    scmp_filter_ctx ctx = NULL;
    int ret = 0;
    ctx = seccomp_init(SCMP_ACT_ALLOW);  // default action: Allow
    /* kernel disallows TSYNC and NOTIFY in one filter unless we
     * have the TSYNC_ESRCH flag */
    // ret = seccomp_attr_set(ctx, SCMP_FLTATR_CTL_TSYNC, 1);
    ret = seccomp_rule_add(ctx, SCMP_ACT_NOTIFY, SCMP_SYS(getpid), 0, NULL);
    ret = seccomp_load(ctx);
    int notify_fd = seccomp_notify_fd(ctx);
    return notify_fd;
}

int test_seccomp_notify()
{
    int fd = set_seccomp_notify();

    struct seccomp_notif *req = NULL;
    struct seccomp_notif_resp *resp = NULL;
    int ret = 0;
    int status = 0;
    pid_t f_pid = fork();
    if (f_pid == 0)
    {
        // child
        pid_t c_pid = getpid();
        printf("Child pid from child: %d\n", c_pid);
        exit(0);
    }

    // parent
    printf("Child pid from parent: %d\n", f_pid);
    seccomp_notify_alloc(&req, &resp);

    seccomp_notify_receive(fd, req); // TODO validate syscall nr
    (void)seccomp_notify_id_valid(fd, req->id); // TODO validate fd and req->id

    resp->id = req->id;
    resp->val = 11111;
    resp->error = 0;
    resp->flags = 0;
    ret = seccomp_notify_respond(fd, resp);

    (void)waitpid(f_pid, &status, 0); // validate status with WIFEXITED and WEXITSTATUS

OUT:
    close(fd);
    seccomp_notify_free(req, resp);
    // seccomp_release(ctx);
}

int main()
{
    test_seccomp_notify();
    return 0;
}
```

#### 其他

1. 可以看看 libseccomp 的代码，比较简单，本质上就是封装 seccomp / ioctl 两个关键的系统调用
2. 可以使用 strace 跟踪一下进程，关注其中的系统调用和参数

### 参考资料

1. [Seccomp Notify – New Frontiers in Unprivileged Container Development](https://people.kernel.org/brauner/the-seccomp-notifier-new-frontiers-in-unprivileged-container-development)

2. [libseccomp](https://github.com/seccomp/libseccomp)

3. [Introduction to seccomp: BPF linux syscall filter](https://www.cnblogs.com/dream397/p/14209766.html)

4. [libseccomp-test](https://gitee.com/mirrors_addons/libseccomp/blob/main/tests/58-live-tsync_notify.c)

5. [seccomp user-trap.c](https://gitee.com/mirrors/linux/blob/master/samples/seccomp/user-trap.c)

   

