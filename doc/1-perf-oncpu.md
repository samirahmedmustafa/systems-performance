In this lab, I will experiment with perf for checking the on-cpu most impacting processes, this will be demonstrated by using unzip command to inflate large .zip file (19GBs), which will give us enough time to investigate using variety of tools in addition to `perf` utility to check the stacks and which is causing most of the overhead

- Downloaded large zip file
- Executed the zip command comes natively with the system (look at the command attributes)

  ```
  [postgres@pgserver backup]$ file $(which unzip)
  /usr/bin/unzip: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=75192522294cd5688ad436362acfaf737d426d40, for GNU/Linux 3.2.0,     stripped
  ```

Starting the investigation using the native unzip command

  `top` command

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
