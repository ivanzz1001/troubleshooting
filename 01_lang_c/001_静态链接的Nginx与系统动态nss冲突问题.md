
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








