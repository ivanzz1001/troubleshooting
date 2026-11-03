
# “静态链接的Nginx与系统动态NSS/SSSD模块冲突”问题

本文记录一次通过`codex`协助定位“静态链接的Nginx部署后worker持续崩溃”问题的解决。

## 1. 现场情况

- Nginx版本为`1.27.3`，二进制是 statically linked
  
- worker 配置为 `user root;`

- 系统NSS：`group: files sss systemd`

  ```bash
  $ cat /etc/nsswitch.conf | grep "group:"
  group:      files sss systemd
  netgroup:   sss files
  # group:     db files
  ```

- `libnss_sss.so.2` 动态依赖系统`glibc 2.34`

  ```bash
  # ldd /lib64/libnss_sss.so.2
      linux-vdso.so.1 (0x00007ffca799a000)
	    libc.so.6 => /lib64/libc.so.6 (0x00007f68de800000)
	    /lib64/ld-linux-x86-64.so.2 (0x00007f68dea31000)
  ```

- master 不断拉起 worker，worker 随即 SIGFPE
  
- 最近 20 秒约崩溃 7.4 万次，服务显示 active，但没有稳定 worker，实际上不可用。core 风暴已经让 systemd-coredump.socket 进入 trigger-limit-hit，所以现在很多 core 没有保存下来

## 2. 问题定位过程

下面我们来看看是如何沿着“现象 → 信号 → 崩溃指令 → 调用栈 → 系统配置”逐层定位出来”可能是静态链接问题“的。

### 2.1 先确认真正崩溃的是谁

一开始`systemctl status nginx`显示:

```text
Active: active (running)
Main PID: ... nginx: master process
```
表面上 Nginx 是运行状态，但进一步检查进程发现:

- master 一直存在；
- worker PID 不断变化；
- worker 刚创建就消失；
- master 持续重新创建 worker

错误日志中反复出现：
```text
start worker process 741197
worker process 741197 exited on signal 8 (core dumped)
start worker process 741218
worker process 741218 exited on signal 8
```
此外通过`dmesg`也能看到大量如下报错信息：

```text
[48579820.230362] Skipping core dump
[48579820.230454] Skipping core dump
[48579820.230848] Skipping core dump
[48579820.234526] Pid 2419210(nginx) over core_pipe_limit
[48579820.243227] Pid 2419250(nginx) over core_pipe_limit
[48579820.243348] Pid 2419252(nginx) over core_pipe_limit
```
因此结论是:

```text
master 正常
  ↓
worker 启动
  ↓
worker 立即崩溃
  ↓
master 重新拉 worker
  ↓
无限循环
```
这也解释了为什么`systemctl`显示active：systemd 只跟踪 master PID，并不知道所有 worker 都无法存活。

### 2.2 从信号类型缩小范围

执行`coredumpctl list nginx`，看到大量记录:

```text
Signal: SIGFPE
```
`SIGFPE`是算术异常，最常见的是：
- 整数除零；
- 整数除法溢出；
- 某些浮点异常

它不是：
- `SIGSEGV`：非法内存访问；
- `SIGABRT`：主动 abort；
- `SIGKILL`：OOM 或人为 kill。

所以排查方向从“内存越界、配置错误、OOM”收敛为：**worker 初始化过程中存在除法异常**

同时排除了`OOM`：

```bash
journalctl -k | grep -Ei 'oom|killed process'
```
没有发现内核 OOM 记录。

### 2.3 确认崩溃发生在 worker 初始化阶段
所有 worker 都是在启动后几乎立即崩溃，而且没有稳定存活到处理请求阶段。

这意味着问题更可能发生在：

```text
fork worker
→ 设置用户和用户组
→ 初始化事件模块
→ 初始化各 HTTP 模块
→ 进入事件循环
```

而不是：

```text
收到 HTTP 请求
→ upstream
→ gzip
→ SSL
→ 日志处理
```

因此业务流量、location、upstream 等暂时不是首要怀疑对象。

### 2.4 检查二进制类型

执行:
```bash
# file /opt/applications/instances/nginx/10.142.80.17/bin/nginx     
/opt/applications/instances/nginx/10.142.80.17/bin/nginx: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), statically linked, for GNU/Linux 3.2.0, not stripped

# ldd  /opt/applications/instances/nginx/10.142.80.17/bin/nginx
	not a dynamic executable
```
通过上面得到两个重要信息：

1. Nginx 是完全静态链接的；
1. 二进制没有 strip，仍然保留函数符号，可以直接看到 Nginx 函数名。

此时“静态 glibc 与 NSS”只是一个怀疑，还不能直接下结论。

### 2.5 尝试获取 core，发现 core 风暴已压垮 coredump 服务

执行：

```bash
coredumpctl info <PID>
```
只能看到：

```text
Signal: 8 (FPE)
Storage: none
```

检查 systemd-coredump(参看后文附录）：

```text
systemd-coredump.socket: Too many incoming connections
systemd-coredump.socket: Trigger limit hit
Accepted: 28214
Refused: 578
```
也就是说，worker 崩溃速度太快，`systemd-coredump`已经被打到限流失败，没有保存可供直接分析的 core 文件。

现场采样时，错误日志显示最近 20 秒大约产生了 7.4 万条 SIGFPE 退出记录。

所以不能使用传统方式：

```bash
gdb nginx core.xxx
```
必须在线截获一个即将崩溃的 worker。

### 2.6 先静态排查 Nginx 自身的除法指令

因为二进制没有 strip，可以扫描 Nginx 函数中的`div`和`idiv`指令：

```bash
# objdump -d -M intel nginx |
awk '
/^[0-9a-f]+ <[^>]+>:/ { fn=$0 }
/[[:space:]](idiv|div)[[:space:]]/ {
    if (fn ~ /<ngx_/) {
        print fn
        print $0
    }
}'

...
000000000035af00 <ngx_event_process_init>:
  35b1a2:	48 f7 f1             	div    rcx
000000000035af00 <ngx_event_process_init>:
  35b1ba:	48 f7 f1             	div    rcx
000000000035af00 <ngx_event_process_init>:
  35b1d9:	48 f7 f1             	div    rcx
000000000035af00 <ngx_event_process_init>:
  35b1f1:	48 f7 f1             	div    rcx
...
```

worker 初始化阶段比较可疑的函数包括:

```text
ngx_event_process_init
ngx_http_userid_init_worker
```
进一步反汇编：

```bash
objdump -d -M intel \
  --start-address=0x35af00 \
  --stop-address=0x35b300 nginx
```
发现`ngx_event_process_init`中的除数固定是：

```text
mov ecx, 0x3e8    # 1000
div rcx
```

ngx_http_userid_init_worker 中则是：

```text
mov ecx, 0x14     # 20
div rcx
```

这两个除数都不可能为零，因此排除了最明显的 Nginx 自身除法路径。

### 2.7 用 GDB 跟踪下一个 worker
因为 worker 生命周期非常短，直接执行：

```bash
gdb -p <worker-pid>
```
基本来不及。

所以采用的办法是：
1. 短暂附加 master；
2. 设置 GDB 跟随下一次 fork 的子进程；
3. fork 后立即脱离 master；
4. 只调试新 worker；
5. 在 SIGFPE 发生时停止。

核心命令是:

```bash
gdb -q -nx -batch \
  -ex 'set follow-fork-mode child' \
  -ex 'set detach-on-fork on' \
  -ex 'handle SIGFPE stop print nopass' \
  -ex "attach $master_pid" \
  -ex continue \
  -ex 'bt full' \
  -ex 'info registers rip rax rbx rcx rdx rsi rdi rbp rsp' \
  -ex 'x/16i $rip-24' \
  -ex detach
```

GDB 成功跟到了新 worker：

```text
Attaching after process ... fork to child process ...
Detaching after fork from parent process ...
```
所以 master 只被短暂暂停，fork 后就脱离调试器。

### 2.8 抓到确切崩溃指令
GDB 停止的位置是：

```text
Thread "nginx" received signal SIGFPE
__libc_early_init() from /lib64/libc.so.6
```

寄存器：
```text
rsi = 0
rip = __libc_early_init+137
```

对应指令：

```text
div %rsi
```

因此这不是推测，而是直接看到：

```text
除数寄存器 rsi = 0
执行 div %rsi
触发 SIGFPE
```

而且崩溃位置不在 Nginx 自身，而是在系统动态库：

```text
/lib64/libc.so.6
```

这非常关键：前面已经确认 Nginx 是静态链接的，为什么静态 Nginx 进程里又出现了系统动态 libc.so.6？

答案只能从完整调用栈中找。

### 2.9 完整调用栈给出了 NSS/SSSD 路径
抓到的栈是：

```text
#0  __libc_early_init
#1  dl_open_worker_begin
#2  _dl_catch_exception
#3  dl_open_worker
#4  _dl_catch_exception
#5  _dl_open
#6  do_dlopen
#7  _dl_catch_exception
#8  _dl_catch_error
#9  __libc_dlopen_mode
#10 module_load
#11 __nss_module_get_function
#12 internal_getgrouplist
#13 initgroups
#14 ngx_worker_process_init
#15 ngx_worker_process_cycle
#16 ngx_spawn_process
#17 ngx_reap_children
#18 ngx_master_process_cycle
#19 main
```

从下往上读：

```text
Nginx 创建 worker
→ ngx_worker_process_init
→ initgroups
→ 查询用户补充组
→ NSS 加载模块
→ dlopen 动态库
→ 动态 libc 初始化
→ 除零 SIGFPE
```
至此已经确定：**崩溃是由 initgroups() 触发的 NSS 动态加载造成的**。

### 2.10 对照 Nginx 和系统配置

继续检查 Nginx worker 用户：

```text
user root;
```

Nginx master 是`root`，因此初始化 worker 时仍会执行类似：

```text
setgid(...);
initgroups("root", ...);
setuid(...);
```
即使目标用户也是 root，Nginx 依然会查询 root 的补充用户组。

查看系统 NSS 配置：

```text
passwd: files sss systemd
group:  files sss systemd
```

当没有单独配置 `initgroups:` 时，补充用户组查询会使用 `group:` 的数据源。

所以实际查询过程是：

```text
/etc/group
→ SSSD
→ systemd
```


检查 SSSD NSS 模块：

```bash
file /lib64/libnss_sss.so.2
ldd  /lib64/libnss_sss.so.2
```

结果：

```text
libnss_sss.so.2: dynamically linked
libc.so.6 => /lib64/libc.so.6
```
这就把所有证据闭环了：

```text
静态 Nginx
内置一套静态 glibc
        ↓
initgroups("root")
        ↓
NSS 根据 nsswitch.conf 进入 sss
        ↓
dlopen(libnss_sss.so.2)
        ↓
libnss_sss.so.2 依赖宿主机动态 libc.so.6
        ↓
动态 libc 在静态 glibc 进程中初始化
        ↓
__libc_early_init 内部状态异常
        ↓
div 0
        ↓
SIGFPE
```

### 2.11 为什么判断不是普通的 SSSD 服务故障

如果只是 SSSD 服务不可达，正常表现应该是：

```text
initgroups 返回失败或超时
```

但这里不是查询失败，而是在加载`libnss_sss.so.2` 及其依赖时，直接进入动态 `loader` 和 `__libc_early_init()`，并在动态 libc 初始化过程中发生除零。

因此：

- 停止`sssd.service`不一定有用；
- 网络是否能连接域控也不是核心；
- 问题发生在真正发起 SSSD 查询之前；
- 关键是加载 SSSD 动态 NSS 模块这个动作本身。

### 2.12 为什么一开始看起来像“worker 正常启动”

master 会先记录：

```text
start worker process 12345
```

之后子进程才执行 ngx_worker_process_init()。

所以时间顺序是：
```text
master fork
→ master 记录“start worker process”
→ ps 短暂看到 worker
→ worker 执行 initgroups
→ SIGFPE
```

主机日志显示：

```text
17:55:03 systemd: Started nginx
17:55:04 大量 worker SIGFPE
```

因此“能看到 worker PID”和“日志出现 start worker process”只能证明 fork 成功，不能证明 worker 完成初始化并进入事件循环.

### 2.13 结论的证据强度

可以把证据分成三层：

1. 直接事实
   - worker 收到 SIGFPE
   - 崩溃指令是 div %rsi
   - rsi=0
   - 崩溃位置是动态 /lib64/libc.so.6::__libc_early_init
   - 调用栈明确经过 initgroups → NSS module_load
2. 配置事实
   - Nginx 是静态链接
   - worker 配置为 root
   - NSS group 包含 sss
   - libnss_sss.so.2 是动态库并依赖系统 libc
3. 由上述事实得到的结论
   - 静态 Nginx 在 worker 初始化时调用 initgroups
   - NSS 动态加载 SSSD
   - SSSD 引入系统动态 libc
   - 动态 libc 初始化异常导致除零

所以“静态链接的 Nginx 与系统动态 NSS/SSSD 模块冲突”不是只根据 file nginx 得出的经验判断，而是由现场 GDB 调用栈和崩溃指令直接定位出来的。


## 3. 修复方案

### 3.1 临时修复方案

临时修复建议：增加独立的`initgroups: files`，让 worker 初始化 root 用户组时只读本地`/etc/group`，不再加载`libnss_sss.so.2`

1）**最快止血方案**


```bash
# cp -a /etc/nsswitch.conf \
  /etc/nsswitch.conf.bak-nginx-core-20261025

# printf '\n# TEMP workaround for static nginx core\ninitgroups: files\n' \
  >> /etc/nsswitch.conf

# getent initgroups root
```

确认输出包含 root 且命令不报错，然后在`nginx.service`恢复后执行：

```bash
# /opt/applications/instances/nginx/10.142.80.17/bin/nginx \
  -e /opt/applications/logs/nginx/nginx/error.log \
  -c /opt/applications/instances/nginx/10.142.80.17/conf/nginx.conf \
  -t

# systemctl daemon-reload
# systemctl reset-failed systemd-coredump.socket
# systemctl start systemd-coredump.socket
# systemctl restart nginx
```

启动后检查：

```bash
# systemctl --no-pager status nginx

# master_pid=$(systemctl show nginx -p MainPID --value)
# ps --ppid "$master_pid" -o pid,ppid,stat,etimes,cmd

# tail -n 50 /opt/applications/logs/nginx/nginx/error.log
```

应该能看到稳定存在的 worker，不再出现新的：

```text
exited on signal 8
```

2）**影响范围**

这一行只改变“为进程计算补充用户组”的查询源：

```text
initgroups: files
```
普通的 `passwd:` 和 `group:` 查询仍保持 files sss systemd。但新启动的域用户进程将暂时拿不到 SSSD 提供的补充组，可能影响基于域用户组的 sudo、文件权限和登录授权。因此只能作为短期止血。参看[Linux nsswitch.conf 手册](https://www.man7.org/linux/man-pages/man5/nsswitch.conf.5.html)

宿主机使用`glibc 2.34`，修改 `nsswitch.conf` 后会自动重新加载，不需要重启机器。

3）**回滚**

部署动态链接版 Nginx 后：

```bash
# cp -a /etc/nsswitch.conf.bak-nginx-core-20261025 \
  /etc/nsswitch.conf

# systemctl restart nginx
```
不要把“停止 SSSD”当成临时修复：即使 SSSD 服务停止，glibc 仍可能先加载 libnss_sss.so.2，core 依然可能发生。

### 3.2 终极修复方案

1）**方式1： 重新构建 Nginx，让系统 glibc 动态链接；同时让 worker 使用专用低权限账户**

2）**方式2：保持使用静态链接，让nginx加载正确版本的libc**

```bash
/opt/applications/instances/nginx/10.142.80.17/lib/ld-linux-x86-64.so.2 --library-path /opt/applications/instances/nginx/10.142.80.17/lib /opt/applications/instances/nginx/10.142.80.17/bin/nginx_with_libc -e /opt/applications/logs/nginx/nginx/error.log -c /opt/applications/instances/nginx/10.142.80.17/conf/nginx.conf
```

## 4. 附录

### 4.1 附录1：如何理解“静态链接的 Nginx 与系统动态 NSS/SSSD 模块冲突”？

简单说：这个 Nginx 把一套 libc“打包进了自己”，但启动 worker 时又临时加载了系统里的另一套 libc，两套运行库混到同一个进程里，最终把内部状态弄坏了。

过程是：

```text
静态 Nginx 启动 worker
  ↓
Nginx 要初始化 root 用户所属的用户组
  ↓
调用 initgroups("root")
  ↓
系统按 nsswitch.conf 查询：
files → sss → systemd
  ↓
加载 SSSD 的 libnss_sss.so.2
  ↓
该模块依赖系统动态 libc.so.6
  ↓
进程里出现“内置 libc + 系统 libc”混用
  ↓
系统 libc 初始化时读到异常值，发生除零
  ↓
worker 收到 SIGFPE 并 core
```

“静态链接”本来的意思是：Nginx 构建时把 libc 等依赖直接装进自己的二进制，不依赖系统共享库。

但 glibc 有一个特殊机制：用户名、用户组、DNS 等查询会根据 `/etc/nsswitch.conf` 动态加载插件。当前机器配置了：

```text
group: files sss systemd
```

> #### 理解 nsswitch.conf 是什么
> 
> NSS（Name Service Switch）是 glibc 提供的一套机制，把"查用户/组/主机名"这类操作从具体数据源中解耦。程序调用 getgrnam()、getgrgid() 等函数时，glibc 会读 nsswitch.conf，按配置的顺序逐个询问各个 NSS 模块，直到得到答案。

因此，即便 Nginx 是静态编译的，调用 `initgroups()` 时仍会尝试加载动态的 `libnss_sss.so.2`。这个 SSSD 模块又需要系统的动态 libc，于是发生冲突。

为什么一直 core：

- master 进程能正常启动；
- master 创建 worker；
- worker 初始化用户组时崩溃；
- master 发现 worker 死亡，再创建一个；
- 新 worker 又走到同一位置崩溃。

所以形成无限循环。

这不是请求流量触发的，也不是某个 location/upstream 配错了。最可靠的解决方式是重新构建或部署一个**动态链接系统 glibc 的 Nginx**。临时绕开 SSSD 查询也可以止住崩溃，但会影响系统的域用户和用户组解析，风险更大。



### 4.2 附录2： 如何检查systemd-coredump？

检查 socket 状态：

```bash
systemctl --no-pager --full status systemd-coredump.socket
```

重点看：

```text
Active: active (listening)
```

如果出现：

```text
Active: failed
Result: trigger-limit-hit
Too many incoming connections
```

说明 core 产生过快，coredump 服务已被限流打挂。

查看相关单元：

```bash
systemctl list-units --all 'systemd-coredump*'
```

查看最近日志：

```bash
journalctl -u systemd-coredump.socket \
  --since '-1 hour' \
  --no-pager
```

查看内核及 coredump 相关错误：

```bash
journalctl --since '-1 hour' --no-pager |
grep -Ei 'coredump|core dump|trigger-limit|too many incoming'
```

查看 Nginx 的 core 记录：

```bash
coredumpctl --no-pager list nginx
```

查看某个 core 的详细信息：

```bash
coredumpctl --no-pager info <PID>
```

例如：

```bash
coredumpctl --no-pager info 1953731
```

重点关注：

```text
Signal:
Executable:
Storage:
Message:
```

如果是：

```text
Signal: 8 (FPE)
Storage: none
```

说明记录到了崩溃元数据，但 core 文件本体没有保存。

查看有效配置：

```bash
systemd-analyze cat-config systemd/coredump.conf
```

查看 core 文件目录：

```bash
du -sh /var/lib/systemd/coredump 2>/dev/null
ls -lh /var/lib/systemd/coredump 2>/dev/null | tail
```

如果 socket 已经因为 core 风暴失败，在先停止崩溃循环之后，可以恢复它：

```bash
systemctl reset-failed systemd-coredump.socket
systemctl start systemd-coredump.socket
systemctl --no-pager status systemd-coredump.socket
```

不要在 Nginx 仍高速 core 时反复启动它，否则很快会再次进入 `trigger-limit-hit`。
