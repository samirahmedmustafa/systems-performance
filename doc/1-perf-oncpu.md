In this lab, using 2 processors, 4GBs memory virtual machine, I will experiment with perf for checking the on-cpu most impacting processes, this will be demonstrated by using unzip command to inflate large .zip file (19GBs), which will give us enough time to investigate using variety of tools in addition to `perf` utility to check the stacks and which is causing most of the overhead

- Downloaded large zip file
- Executed the zip command comes natively with the system (look at the command attributes)

  ```
  [postgres@pgserver backup]$ file $(which unzip)
  /usr/bin/unzip: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=75192522294cd5688ad436362acfaf737d426d40, for GNU/Linux 3.2.0,     stripped
  ```

Starting the investigation by executing the native `unzip` command with the 19GBs zipped file

1. `top` command output

  ```
  [root@pgserver FlameGraph]# top
  top - 17:43:27 up 4 days, 13 min,  4 users,  load average: 0.23, 0.07, 0.06
  Tasks: 191 total,   2 running, 189 sleeping,   0 stopped,   0 zombie
  %Cpu(s): 38.7 us, 12.9 sy,  0.0 ni, 48.4 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
  MiB Mem :   3658.6 total,    140.4 free,    680.1 used,   3264.4 buff/cache
  MiB Swap:   1024.0 total,    932.2 free,     91.8 used.   2978.5 avail Mem
  
      PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
      54650 postgres  20   0    5296   4224   2048 R  93.8   0.1   0:15.09 **unzip**
        1 root      20   0  175088  11240   6100 S   0.0   0.3   0:07.99 systemd
        2 root      20   0       0      0      0 S   0.0   0.0   0:00.57 kthreadd
        3 root      20   0       0      0      0 S   0.0   0.0   0:00.00 pool_workqueue_
        4 root       0 -20       0      0      0 I   0.0   0.0   0:00.00 kworker/R-rcu_g
  ```

  Observations:
  - Unzip command using 93% of the cpu
  - Resident set size and Virtual memory size around 4MBs and 5MBs respectively (memory consumption)
  - Process is in the Running state
  - Priority 20

2. `vmstat` command output

```
[root@pgserver FlameGraph]# vmstat -Sm 1
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0     96    239      0   3339    0    0   111   167   54   14 11  0 89  0  0
 1  0     96    148      0   3431    0    0 122408  2040 3507 4906 40 12 49  0  0
 1  0     96    183      0   3395    0    0 97800 308039 4395 6016 40 16 43  1  0
 1  0     96    135      0   3442    0    0 76940  1067 3445 4657 40 11 48  0  0
 1  0     96    183      0   3392    0    0 84864 307725 4247 5599 41 15 44  1  0
```

  Observations:
  - Utilization around 50% (40% user time + 11% system time), probably only one cpu is utilized otherwise it would be close to 100%
  - Average number interrupts and context switching 4000 and 5000 respectively, that could be due to disk/network or lockings
  - There is a continuous runnable process on the queue waiting (although I suspect only 1 cpu is utilized)
  - 3.5 GBs free memory (free + cache)
  - no swapping
  - minimal wait time on disks

3. Checking CPUs status

   ```
   [root@pgserver FlameGraph]# sar -P ALL 1
    Linux 5.14.0-503.14.1.el9_5.x86_64 (pgserver)   11/28/2025      _x86_64_        (2 CPU)
    
    06:09:50 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
    06:09:51 PM     all     40.00      0.00     17.00      1.00      0.00     42.00
    06:09:51 PM       0     55.45      0.00     23.76      1.98      0.00     18.81
    06:09:51 PM       1     24.24      0.00     10.10      0.00      0.00     65.66
    
    06:09:51 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
    06:09:52 PM     all     39.39      0.00     13.64      0.51      0.00     46.46
    06:09:52 PM       0     76.77      0.00     22.22      1.01      0.00      0.00
    06:09:52 PM       1      2.02      0.00      5.05      0.00      0.00     92.93
    
    06:09:52 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
    06:09:53 PM     all     40.50      0.00     14.00      0.50      0.00     45.00
    06:09:53 PM       0     56.00      0.00     19.00      1.00      0.00     24.00
    06:09:53 PM       1     25.00      0.00      9.00      0.00      0.00     66.00
    
    06:09:53 PM     CPU     %user     %nice   %system   %iowait    %steal     %idle
    06:09:54 PM     all     38.50      0.00     16.50      1.00      0.00     44.00
    06:09:54 PM       0     76.00      0.00     23.00      1.00      0.00      0.00
    06:09:54 PM       1      1.00      0.00     10.00      1.00      0.00     88.00
```
```
  Observations:
  - Minimal IO wait
  - CPU 0 is loaded while CPU 1 is idle most of the time (less than 20% CPU1 and more than 60% CPU0)

4. Check network status
   
   ```
   sar -n DEV 1
    Linux 5.14.0-503.14.1.el9_5.x86_64 (pgserver)   11/28/2025      _x86_64_        (2 CPU)
    
    06:09:20 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
    06:09:21 PM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    06:09:21 PM    enp1s0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    06:09:21 PM    enp9s0    599.00   1199.00     38.61    158.18      0.00      0.00      0.00      0.00
    
    06:09:21 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
    06:09:22 PM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    06:09:22 PM    enp1s0      1.00      0.00      0.05      0.00      0.00      0.00      0.00      0.00
    06:09:22 PM    enp9s0    667.00   1330.00     43.01    175.82      0.00      0.00      0.00      0.00
    
    06:09:22 PM     IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
    06:09:23 PM        lo      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    06:09:23 PM    enp1s0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    06:09:23 PM    enp9s0    717.00   1433.00     46.25    189.93      0.00      0.00      0.00      0.00
   ```

   Observations:
   - Very mimimum network utilization

5. Check disks status

   ```
   [root@pgserver FlameGraph]# sar -d 1
    Linux 5.14.0-503.14.1.el9_5.x86_64 (pgserver)   11/28/2025      _x86_64_        (2 CPU)
    
    05:43:15 PM       DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
    05:43:16 PM       vda      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    05:43:16 PM       vdb    199.00 133264.00   1449.00      0.00    676.95      0.25      1.18      5.90
    05:43:16 PM      dm-0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    05:43:16 PM      dm-1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    05:43:16 PM      dm-2    119.00 133264.00   1449.00      0.00   1132.04      0.12      0.99      5.90
    
    05:43:16 PM       DEV       tps     rkB/s     wkB/s     dkB/s   areq-sz    aqu-sz     await     %util
    05:43:17 PM       vda      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    05:43:17 PM       vdb    503.00 119752.00 310476.00      0.00    855.32      2.90      5.68     17.00
    05:43:17 PM      dm-0      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    05:43:17 PM      dm-1      0.00      0.00      0.00      0.00      0.00      0.00      0.00      0.00
    05:43:17 PM      dm-2   1626.00 119752.00 310476.00      0.00    264.59     11.06      6.80     17.00
   ```

   Observations:
   - Minimum disk utilization 17%
   
You can enable number of threads to check whether the process is multithreaded or no. For enabling number of threads in `top` command (press f to get the list of options, then go down to `nTH` enable it using space then `ESC` to go back to the top screen)

```
    top - 18:25:07 up 4 days, 55 min,  3 users,  load average: 0.22, 0.10, 0.14
    Tasks: 183 total,   4 running, 179 sleeping,   0 stopped,   0 zombie
    %Cpu(s): 40.0 us, 14.7 sy,  0.0 ni, 43.9 id,  1.0 wa,  0.0 hi,  0.3 si,  0.0 st
    MiB Mem :   3658.6 total,    123.9 free,    676.1 used,   3283.4 buff/cache
    MiB Swap:   1024.0 total,    932.2 free,     91.8 used.   2982.5 avail Mem


    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                            nTH
    55025 postgres  20   0    5296   4096   1920 R  96.7   0.1   0:15.87 unzip                                                                                1
       54 root      20   0       0      0      0 S   3.7   0.0   1:07.93 kswapd0                                                                              1
    29581 ansible   20   0   20656   6512   5248 S   3.3   0.2   0:34.74 sshd                                                                                 1
    55000 root      20   0       0      0      0 I   2.3   0.0   0:00.40 kworker/u9:2-events_unbound                                                          1
    47114 root      20   0       0      0      0 R   1.7   0.0   0:11.10 kworker/u10:3-events_unbound                                                         1
    55017 root      20   0       0      0      0 I   1.0   0.0   0:00.08 kworker/1:1-events_freezable_pwr_ef                                                  1
    54951 root      20   0       0      0      0 I   0.7   0.0   0:00.64 kworker/0:8-events_power_efficient                                                   1
```

6. Checking On-cpu using perf command

```
perf record -F 99 -a -g -- sleep 10
perf script --header > oncpuout.stacks
git clone https://github.com/brendangregg/FlameGraph; cd FlameGraph
./stackcollapse-perf.pl < ../oncpuout.stacks | ./flamegraph.pl --hash > /tmp/oncpu_stripped.svg
```

Observations:

- According to the below flamegraph snapshot the unzip command is taking 86% of the CPU time (but that is not showing stacktrace of the internal functions inside `unzip` which are causing the 86%)
<img width="1920" height="1140" alt="image" src="https://github.com/user-attachments/assets/7549855e-391f-4f30-a0e2-3ce2b986d656" />


7. I download unzip source code and managed to compile it introducing the options `-g -fno-omit-frame-pointer` (as in [unzip rebuild](unzip-rebuild.md)) and got the below flamegraph
   <img width="1920" height="1140" alt="image" src="https://github.com/user-attachments/assets/581bc961-4223-4d42-83ee-ecb28ed6888e" />


That's the recompile way of getting the flame graph stack complete info, the other one is using the debuginfo
