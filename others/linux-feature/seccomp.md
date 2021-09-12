### seccomp

### 简介

之前只知道，Docker 使用的核心技术是 namespace 和 cgroup。这几天，发现一个新的技术点：seccomp。seccomp 是 secure computing mode 的缩写，意为构造一个安全的计算环境 -- 限制服务对系统调用（syscal）的使用。

seccomp 主要经历两个阶段：

1. 从内核2.6.23引入的，通过 prctl 设置，设置之后，该进程只支持以下四个基本的系统调用：read / write / _exit / sigreturn。由于限制太死了，并不是很实用。
2. 在内核3.5，使用基于 BPF 的系统调用过滤功能，渐渐强大起来。支持指定 filter 以及对于的 action。不过，实际编写 BPF 还是比较麻烦，所以就有了 libseccomp 库，使用就比较方便了。

### 过滤条件 filter

1. 系统调用：指定特定的系统调用作为过滤条件，匹配上则执行对应的动作

   获取某个系统调用的编号：/usr/include/x86_64-linux-gnu/asm/unistd_64.h

2. 系统调用的参数：可以对于同个系统调用，根据参数不同指定不同的动作。比如只允许 dup 标准输出，不能 dup 标准错误等

### 执行动作 action

1. SCMP_ACT_KILL
2. SCMP_ACT_KILL_PROCESS：kill 掉整个进程树
3. SCMP_ACT_TRAP：抛出 SIGSYS 信号
4. SCMP_ACT_ERRNO：返回指定错误码
5. SCMP_ACT_TRACE
6. SCMP_ACT_LOG：syslog 打个日志，其他的不变
7. SCMP_ACT_ALLOW：正常调用
8. SCMP_ACT_NOTIFY：触发通知

以上通过 man seccomp_rule_add 可以获取相关信息。



这里面可能最有意思的，应该是 ACT_NOTIFY 了，触发通知，可以做很多事情。而且，如果使用 notify，那么调用对应系统调用的进程会被 block 住，直到监控进程有对应的 seccomp_notify_respond。

```c
/* test for linux-feature
   secure computing, filter syscall
 */
#include <stdio.h>          // for printf
#include <sys/prctl.h>      // for prctl
#include <linux/seccomp.h>  // for seccomp constant
#include <unistd.h>         // for dup / fork / access
#include <sys/types.h>      // for pid_t
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

    (void)access("/test_file_path", F_OK);
    printf("YOU SHOULD NOT SEE ME!!!\n");

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

    fd = open("/etc/hosts", O_RDONLY);
    printf("open(%d) file /etc/hosts fd: %d, errno: %d\n", SCMP_SYS(openat), fd, errno);

    ret = access("/etc/hosts", F_OK);
    printf("access(%d) file /etc/hosts ret: %d\n", SCMP_SYS(access), ret);

    (void)dup(1);
    printf("dup(%d) fd 1\n", SCMP_SYS(dup));

    (void)dup(2);
    printf("YOU SHOULD NOT SEE ME!!!\n");}

int main()
{
    // test_seccomp_strict();
    test_seccomp_filter();
    return 0;
}

```



### 参考资料

1. [Seccomp Notify – New Frontiers in Unprivileged Container Development](https://people.kernel.org/brauner/the-seccomp-notifier-new-frontiers-in-unprivileged-container-development)

