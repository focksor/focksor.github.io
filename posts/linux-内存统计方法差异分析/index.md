# Linux 内存统计方法差异分析



## 概述

本文将探索 Linux 下不同内存统计方法的差异，以及描述相应方法的适用场景。

你可能想要知道 top, ps, systemctl status, smem 等工具或命令显示的内存用量分别有什么含义，其显示的数字又为什么各有差异，本文将进行深入分析，带你了解其中的奥妙，并让你清楚应该在什么场景下应该使用哪个指标。

## 背景

笔者在分析 Linux 机器上的内存使用情况时，发现一个奇怪的现象：`systemctl` 和 `ps` 的内存（RSS）数据对不上，两者有时候会无法形成逻辑上的关联关系。

例如，在分析 Nginx 应用的内存使用情况时，笔者观察到了如下的数据：
```shell
# uname -a
Linux firewall 5.10.204-g39484bf51955 #1 SMP Tue Jun 10 01:12:29 CST 2025 x86_64 GNU/Linux
# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) since Tue 2025-06-10 09:11:03 CST; 2h 31min ago
    Process: 3954 ExecReload=/usr/sbin/nginx -g daemon on; master_process on; -s reload (code=exited, status=0/SUCCESS)
   Main PID: 665 (nginx)
      Tasks: 3 (limit: 5617)
     Memory: 9.9M
     CGroup: /system.slice/nginx.service
             ├─ 665 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             └─3958 nginx: worker process
# ps aux | head -1 && ps aux | grep nginx
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         665  0.0  0.0 113068  8820 ?        Ss   09:11   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data    3958  0.0  0.0 182940 12832 ?        Sl   09:12   0:00 nginx: worker process
```

如上我们可以从 `systemctl status nginx` 中看到，nginx 启动了，分别是一个 master 和一个 worker，两者的内存一共占用了 9.9M。而当我们使用 ps 指令去查看时，发现两个进程的 RSS 内存使用量分别是 8820 和 12832，即总共用了 8820+12832=21652kB=21.14MB。明显大于 `systemctl status` 显示的内存用量。

那有人就要说了，nginx 的 worker 很明显是从 master 中 fork 出来的，所以两个进程的内存有一部分是共享的（shared and COW），所以会比两个进程单独显示的用量总数大，很合理吧？

一开始笔者也是这么认为的，然而随后便发现这个说法经不起推敲：worker 的总内存用量 12832=12.5MB，大于 `systemctl status` 中显示的内存总量  9.9M。就算 worker 共享了 master 的全部内存，那也不能比总量大吧？

暂时没想明白这是怎么算出来的，笔者随后又去看了另一台机器（nginx 配置相同但机器硬件和系统不同），结果看到了更离谱的数据：
```shell
# uname -a
Linux firewall 4.14.76-gd85b09009 #1 SMP PREEMPT Wed May 28 16:22:58 CST 2025 aarch64 GNU/Linux
# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) (thawing) since Mon 2025-06-09 16:14:51 CST; 19h ago
   Main PID: 7401 (nginx)
      Tasks: 3 (limit: 4629)
     Memory: 26.5M
     CGroup: /system.slice/nginx.service
             ├─7401 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             └─7403 nginx: worker process
# ps aux | head -1 && ps aux | grep nginx
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root        7401  0.0  0.0  43328  2056 ?        Ss   Jun09   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data    7403  0.0  0.2 121796 11524 ?        Sl   Jun09   0:15 nginx: worker process
```

可以看到，同样是 nginx，其 master 和 worker 的 RSS 内存占用分别是 2056 和 11524，总量是 2056+11524=13580kB=13.26MB，甚至比 `systemctl status nginx` 上显示的 26.5M 更小？

作为一个严谨的程序员，笔者想要找出其背后的原理，通过统一的理论同时解释上面两组数据——更重要的是，让我可以知道究竟哪个数据才能真实地表达对应应用程序的内存情况。


## 理论分析

> [!info]
> 本文档分析过程基于 [Linux kernel 4.14](https://github.com/torvalds/linux/tree/v4.14) 版本


### 命令数据来源

#### ps RSS 数据来源
阅读 [源码](https://gitlab.com/procps-ng/procps/-/tree/v3.3.15?ref_type=tags)，不难看出 `ps` 指令的 RSS 内存用量直接读取自 `/proc/<pid>/status` 中的 `VmRSS` 字段。如下也可以看到两个数据是完全一样的：
```shell
# ps aux | head -1 && ps aux | grep nginx
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      129137  0.0  0.0  43312  2340 ?        Ss   Jun11   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data  129139  0.0  0.3 121700 12148 ?        Sl   Jun11   0:13 nginx: worker process
# cat /proc/129137/status | grep VmRSS
VmRSS:      2340 kB
# cat /proc/129139/status | grep VmRSS
VmRSS:     12148 kB
```

> [!note]
> 源码定位：[main()](https://gitlab.com/procps-ng/procps/-/blame/v3.3.15/ps/display.c?ref_type=tags#L673) -> [simple_spew](https://gitlab.com/procps-ng/procps/-/blame/v3.3.15/ps/display.c?ref_type=tags#L375) -> [show_one_proc](https://gitlab.com/procps-ng/procps/-/blame/v3.3.15/ps/output.c?ref_type=tags#L2054) -> [format_array](https://gitlab.com/procps-ng/procps/-/blob/v3.3.15/ps/output.c?ref_type=tags#L1649) -> [pr_rss](https://gitlab.com/procps-ng/procps/-/blob/v3.3.15/ps/output.c?ref_type=tags#L990) -> [status2proc](https://gitlab.com/procps-ng/procps/-/blame/v3.3.15/proc/readproc.c?ref_type=tags#L369)

#### top RSS 数据来源
阅读 [源码](https://gitlab.com/procps-ng/procps/-/tree/v3.3.15?ref_type=tags)，不难发现 `top` 指令的 `RES` 内存用量取自 `/proc/<pid>/statm` 并乘以每页大小。如下也可以看到两个数据是完全一样的：
```shell
# top -p 129137 -p 129139
   PID USER      PR  NI    VIRT    RES  %CPU  %MEM     TIME+ S COMMAND
129137 root      20   0   43312   2340   0.0   0.1   0:00.00 S nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
129139 www-data  20   0  121700  12148   0.0   0.3   0:13.42 S  `- nginx: worker process
# awk '{print $2*4}' /proc/129137/statm
2340
# awk '{print $2*4}' /proc/129139/statm
12148
```
> [!note]
> 源码定位：[main()](https://gitlab.com/procps-ng/procps/-/blob/v3.3.15/top/top.c?ref_type=tags#L6134) -> [frame_make](https://gitlab.com/procps-ng/procps/-/blob/v3.3.15/top/top.c?ref_type=tags#L6088) -> [window_show](https://gitlab.com/procps-ng/procps/-/blob/v3.3.15/top/top.c?ref_type=tags#L5995) -> [task_show](https://gitlab.com/procps-ng/procps/-/blob/v3.3.15/top/top.c?ref_type=tags#L5836) -> [statm2proc](https://gitlab.com/procps-ng/procps/-/blob/v3.3.15/proc/readproc.c?ref_type=tags#L655)

#### smem RSS 数据来源
阅读 [源码](https://selenic.com/repo/smem/file/tip)，可以看到 `smem` 指令的 `RSS` 内存用量取自 `/proc/<pid>/smaps`，是一个进程所有已映射内存区域的 RSS 总和（即也是 `/proc/<pid>/smaps_rollup` 中 `Rss` 字段的值）。如下也可以看到几个数据是完全一样的：
```shell
# smem -P nginx
  PID User     Command                         Swap      USS      PSS      RSS
129137 root     nginx: master process /usr/        4      480     1175     3144
129139 www-data nginx: worker process              4     6540     7537    12352
# awk '/^Rss:/ {rss += $2} END {print rss " kB"}' /proc/129137/smaps
3144 kB
# grep Rss /proc/129137/smaps_rollup
Rss:                3144 kB
# awk '/^Rss:/ {rss += $2} END {print rss " kB"}' /proc/129139/smaps
12352 kB
# grep Rss /proc/129139/smaps_rollup
Rss:               12352 kB
```
> [!note]
> 源码定位：
> [showpids](https://selenic.com/repo/smem/file/98273ce331bb/smem#l290) -> [processtotals](https://selenic.com/repo/smem/file/98273ce331bb/smem#l259) -> [pidtotals](https://selenic.com/repo/smem/file/98273ce331bb/smem#l240) -> [pidmaps](https://selenic.com/repo/smem/file/98273ce331bb/smem#l160)


#### systemctl status RSS 数据来源
阅读 [源码](https://github.com/systemd/systemd/tree/v246)，可以看到 `systemctl status` 的 `Memory` 数据用量取自 `/sys/fs/cgroup/memory/<unit.path>/memory.usage_in_bytes` (cgroup v1)或 `/sys/fs/cgroup/memory/<unit.path>/memory.current`（cgroup v2） 。如下也可以看到两个数据是完全一样的：
```shell
# systemctl show nginx -p MemoryCurrent
MemoryCurrent=10371072
# cat /sys/fs/cgroup/memory/system.slice/nginx.service/memory.usage_in_bytes
10371072
# awk '{printf "%.2f MiB\n", $1/1024/1024}' /sys/fs/cgroup/memory/system.slice/nginx.service/memory.usage_in_bytes
9.89 MiB
# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) (thawing) since Wed 2025-06-11 17:39:02 CST; 4 days ago
   Main PID: 129137 (nginx)
      Tasks: 3 (limit: 4629)
     Memory: 9.8M
     CGroup: /system.slice/nginx.service
             ├─129137 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             └─129139 nginx: worker process
```

> [!note]
> 源码定位：[print_status_info](https://github.com/systemd/systemd/blob/ae366f3acbc1a45504e9875099b17a7e1a221d03/src/systemctl/systemctl.c#L4457) -> [status_map](https://github.com/systemd/systemd/blob/ae366f3acbc1a45504e9875099b17a7e1a221d03/src/systemctl/systemctl.c#L5502) -> [bus_unit_cgroup_vtable](https://github.com/systemd/systemd/blob/ae366f3acbc1a45504e9875099b17a7e1a221d03/src/core/dbus-unit.c#L1530) -> [property_get_current_memory](https://github.com/systemd/systemd/blob/ae366f3acbc1a45504e9875099b17a7e1a221d03/src/core/dbus-unit.c#L1082) -> [unit_get_memory_current](https://github.com/systemd/systemd/blob/ae366f3acbc1a45504e9875099b17a7e1a221d03/src/core/cgroup.c#L3165) ->cg_get_attribute_as_uint64


### 内核数据来源
在上个小节对各命令的数据来源分析中，我们可以看到每个命令的数据其实是来源于不同的文件，而这些文件是由 Linux 内核创建的虚拟文件，反映了程序的运行状态。下面是这些文件相应字段的定义：

| 文件                                                       | 字段     | 使用文件数据的命令 | 说明                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| ---------------------------------------------------------- | -------- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `/proc/<pid>/status`                                       | VmRSS    | top                | size of memory portions. It contains the three following parts (VmRSS = RssAnon + RssFile + RssShmem)[](https://docs.kernel.org/filesystems/proc.html#:~:text=size%20of%20memory%20portions.%20It%20contains%20the%20three%20following%20parts%20(VmRSS%20%3D%20RssAnon%20%2B%20RssFile%20%2B%20RssShmem))                                                                                                                                                                                                                                                               |
| `/proc/<pid>/statm`                                        | resident | ps                 | (same as VmRSS in status)[](https://docs.kernel.org/filesystems/proc.html#:~:text=(same%20as%20VmRSS%20in%20status))                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `/proc/<pid>/smaps_rollup`                                 | Rss      | smem               | it contains these fields:[](https://docs.kernel.org/filesystems/proc.html#:~:text=it%20contains%20these,Pss_Shmem)<br>- Pss_Anon<br>- Pss_File<br>- Pss_Shmem<br>                                                                                                                                                                                                                                                                                                                                                                                                        |
| `/sys/fs/cgroup/memory/\<unit.path>/memory.usage_in_bytes` | NA       | systemctl status   | 5.5 usage_in_bytes[](https://docs.kernel.org/admin-guide/cgroup-v1/memory.html#usage-in-bytes "Permalink to this heading")<br><br>For efficiency, as other kernel components, memory cgroup uses some optimization to avoid unnecessary cacheline false sharing. usage_in_bytes is affected by the method and doesn’t show ‘exact’ value of memory (and swap) usage, it’s a fuzz value for efficient access. (Of course, when necessary, it’s synchronized.) If you want to know more exact memory usage, you should use RSS+CACHE(+SWAP) value in memory.stat(see 5.2). |

注意到 procfs 的 status 和 statm 完全一致，包含了一个进程中所有 匿名+文件映射+共享内存映射 内存。

而 smaps_rollup 也是同样的定义，按理来说其数值与上面的值一模一样，但是由于统计方法的差异（status/statm 使用计数器进行统计，smaps 的值是遍历 VMZ 计算来的，所以会更精确），smaps_rollup 的值一般会稍大于 status/statm，但是差异不会很大。

> [!quote]
> /proc/pid/statm - memory usage information
> 
> Some of these values are inaccurate because of a kernel-
  internal scalability optimization.  If accurate values are
  required, use _/proc/pid/smaps_ or _/proc/pid/smaps_rollup_
  instead, which are much slower but provide accurate,
  detailed information.[](https://www.man7.org/linux/man-pages/man5/proc_pid_statm.5.html#:~:text=Some%20of%20these,accurate%2C%0A%20%20%20%20%20%20%20%20%20%20%20%20%20%20detailed%20information.)

而对于 cgroupfs，其中的 `memory.usage_in_bytes` 约等于 `memory.stat` 中的 RSS+CACHE。

但是 cgroup 的计数方法与 procfs 中的不一样，其采用的是“首次接触时计费法”，即谁先映射一块共享地址，这块地址的内存开销就算在谁头上。因此，cgroup 的内存统计数据有时候会与实际不同，更多会与服务启动顺序相关（例如，一个服务启动得早，相关动态库的开销就算它上面了，而让其它服务使用相关动态库的服务先启动会降低该服务显示的内存使用量）。

> [!quote]
> 2.3 Shared Page Accounting[](https://docs.kernel.org/admin-guide/cgroup-v1/memory.html#shared-page-accounting "Permalink to this heading")
> 
>  Shared pages are accounted on the basis of the first touch approach. The cgroup that first touches a page is accounted for the page. The principle behind this approach is that a cgroup that aggressively uses a shared page will eventually get charged for it (once it is uncharged from the cgroup that brought it in -- this will happen on memory pressure).

## 实践分析

回到开头的例子，当一个 systemd 服务包含多个进程的时候，为什么有时候 service memory 数值比所有进程的 ps RSS 加起来多，而有时候比单个进程 ps RSS 还要少呢？让我们分情况进行讨论。


### systemctl status 小于 ps/top
当 systemctl status 显示的 Memory 量比其所有子进程通过 ps/top 命令看到的 RSS 量加起来更少，这是很正常的，也是最常见的。

这主要是因为 systemctl 对于共享内存空间（如多进程共享的动态库）等的统计规则是“首次接触时计费”，而 ps/top 的 RSS 统计量是会在每个进程中都计入共享的内存空间。

如下案例，可以看到 systemctl status 的内存用量小于该服务内个进程 RSS 用量的总和：
```shell
# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) (thawing) since Wed 2025-06-11 17:39:02 CST; 6 days ago
   Main PID: 129137 (nginx)
      Tasks: 3 (limit: 4629)
     Memory: 9.9M
     CGroup: /system.slice/nginx.service
             ├─129137 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             └─129139 nginx: worker process
# ps aux | head -1 && ps aux | grep nginx
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root      129137  0.0  0.0  43312  2340 ?        Ss   Jun11   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data  129139  0.0  0.3 121700 12152 ?        Sl   Jun11   0:15 nginx: worker process
```

### systemctl status 大于 ps/top
当 systemctl status 显示的 Memory 量比其所有进程通过 ps/top 命令看到的 RSS 量加起来更多时，这种情况也是很正常的。

上文中我们提到，systemctl status 的数据源于 cgroup 中的 memory.usage_in_bytes，而该文件的用量近似于 memory.stat 中的 RSS+CACHE。这里的 cache 除了常规的共享空间外，还包含了内核为该 cgroup 缓存的文件数据大小，属于内存中的页缓存部分。这部分缓存是操作系统为了提高文件 I/O 性能而保留的，缓存内容是文件的磁盘内容副本。

如下案例，我们可以看到 systemctl status 的内存用量远大于该服务内个进程 RSS 用量的总和：

```shell
# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) (thawing) since Sat 2025-05-24 02:36:41 CST; 3 weeks 3 days ago
    Process: 2083 ExecReload=/usr/sbin/nginx -g daemon on; master_process on; -s reload (code=exited, status=0/SUCCESS)
   Main PID: 685 (nginx)
      Tasks: 3 (limit: 4629)
     Memory: 31.6M
     CGroup: /system.slice/nginx.service
             ├─ 685 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             └─2107 nginx: worker process
# ps aux | head -1 && ps aux | grep nginx
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         685  0.0  0.1  43684  7832 ?        Ss   May24   0:00 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
www-data    2107  0.0  0.3 121748 12720 ?        Sl   May24   2:32 nginx: worker process
```

查看相应的 memory.stat 文件，可以看到其 cache 比例特别高，而 RSS 距离 systemctl status 显示的数据相差甚远：
```shell
# head -2 /sys/fs/cgroup/memory/system.slice/nginx.service/memory.stat
cache 24866816
rss 7581696
# head -2 /sys/fs/cgroup/memory/system.slice/nginx.service/memory.stat | \
> awk '{ printf "%s %.2f MiB\n", $1, $2 / 1024 / 1024 }'
cache 23.71 MiB
rss 7.23 MiB
```

要如何说明这里的 cache 中包含了很多内核缓存的文件数据呢？让我们清理一下缓存试试：
```shell
# echo 3 > /proc/sys/vm/drop_caches
# head -2 /sys/fs/cgroup/memory/system.slice/nginx.service/memory.stat | \
> awk '{ printf "%s %.2f MiB\n", $1, $2 / 1024 / 1024 }'
cache 1.79 MiB
rss 7.23 MiB
# systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
     Active: active (running) (thawing) since Sat 2025-05-24 02:36:41 CST; 3 weeks 3 days ago
    Process: 2083 ExecReload=/usr/sbin/nginx -g daemon on; master_process on; -s reload (code=exited, status=0/SUCCESS)
   Main PID: 685 (nginx)
      Tasks: 3 (limit: 4629)
     Memory: 9.7M
     CGroup: /system.slice/nginx.service
             ├─ 685 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
             └─2107 nginx: worker process
```

可以看到，清理缓存后，cache 一下子少了很多（但是不是全没了，因为还包含共享内存等缓存空间），而 systemctl status 的数据也回到了我们上个案例中的水平，足以验证我们的理论是完全正确的。

### systemctl status 等于 ps/top

那么可能就有人要问了，systemctl status 可能小于 ps/top 的值，也可能大于 ps/top 的值，那能不能基本等于这个值呢？

让我们回忆上面的内容，当 systemd service 启动时，所有加载的数据都在私有内存空间（或首次加载的共享内存空间）的话，由于 ps/top 也会计算到这些信息，这个时候两者统计的 RSS 信息就基本一致了。当然也别忘了 cgroup 的 cache 还附加了内核缓存的文件数据大小，这个也要清理一下。

如此操作之后，systemctl status 的值应该就基本等于 ps/top 的值了，由于统计口径的差别，两者可能会有一定的偏差，但理论上来说差异不会太大。

## 总结

在上面的文档中，我们一共分析了这些命令：ps、top、smem、systemctl status。从这些命令的不同实现方法中，我们也可以看到它们的作者在实现时的思考：
* ps/top 命令数据取自 procfs 中的 statm 和 status，两者数据完全一致，反应了所有映射到一个程序的内存空间的内存总量。
* smem 稍微复杂，分了 USS, PSS, RSS 几个指标，其中：
	* USS 表示一个进程独享的内存量，反应了 kill 掉这个进程能释放的空间
	* PSS 表示一个进程分摊后的负载，是 USS + (共享部分内存空间 / 共享进程数)
	* RSS 与 ps/top 的指标基本相同，仅因为统计口径不同数据稍有偏差
* systemctl status 使用了特殊的“首次接触时计费”统计法，而且还包含了内存缓存的文件信息，反应了“启动这个服务消耗的内存”。

因此，这些命令适用的场景不同，我们在分析系统内存占用时，需要根据具体的需求选择合适的工具：
- **当需要快速了解进程的整体内存映射情况时**，如进行系统资源初步评估或排查进程是否异常占用内存，使用 `ps` 或 `top` 是最直接的选择，它们开销小、信息全面，但无法细化内存来源或共享程度。
- **当需要准确评估某个服务或进程对系统内存的真实负担时**，推荐使用 `smem`。特别是在容器化、多进程共享库广泛使用的系统中，`smem` 提供的 USS/PSS 能够更真实地反映每个进程的“责任内存”，避免因共享库导致的重复计算。
- **当我们希望从系统服务的角度出发，评估每个 systemd 管理的服务对系统资源的消耗时**，`systemctl status` 提供的“首次接触时计费”是一种更贴近服务视角的内存估算方式。它能够包含服务启动时加载的共享库和缓存文件所带来的额外内存开销，对容量规划和资源分配更有参考意义。

最后需要注意的是，不同命令在统计口径上存在差异，有些数据来源于内核快照，有些来源于动态计算，可能在高负载或频繁变化的场景下出现偏差。因此在分析复杂内存问题时，我们可能需要结合多种工具交叉验证，并深入查看 `/proc/<pid>/smaps` 等底层接口以获取更精细的数据支持。

## 资源

1. [procps-ng / procps · GitLab](https://gitlab.com/procps-ng/procps)
2. [systemd/systemd: The systemd System and Service Manager](https://github.com/systemd/systemd)
3. [torvalds/linux at v4.14](https://github.com/torvalds/linux/tree/v4.14)
4. [Memory Resource Controller — The Linux Kernel documentation](https://docs.kernel.org/admin-guide/cgroup-v1/memory.html)
5. [memory management - Linux OS: /proc/[pid]/smaps vs /proc/[pid]/statm - Stack Overflow](https://stackoverflow.com/questions/30744333/linux-os-proc-pid-smaps-vs-proc-pid-statm/30799817#30799817)

