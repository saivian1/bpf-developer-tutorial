## eBPF 入门实践教程：

## origin

origin from:

<https://github.com/iovisor/bcc/blob/master/libbpf-tools/runqlat.bpf.c>

This program summarizes scheduler run queue latency as a histogram, showing
how long tasks spent waiting their turn to run on-CPU.

## Compile and Run

Compile:

```shell
docker run -it -v `pwd`/:/src/ yunwei37/ebpm:latest
```

```console
$ ecc runqlat.bpf.c runqlat.h
Compiling bpf object...
Generating export types...
Packing ebpf object and config into package.json...
```

Run:

```console
$ sudo ecli examples/bpftools/runqlat/package.json -h
Usage: runqlat_bpf [--help] [--version] [--verbose] [--filter_cg] [--targ_per_process] [--targ_per_thread] [--targ_per_pidns] [--targ_ms] [--targ_tgid VAR]

A simple eBPF program

Optional arguments:
  -h, --help            shows help message and exits 
  -v, --version         prints version information and exits 
  --verbose             prints libbpf debug information 
  --filter_cg           set value of bool variable filter_cg 
  --targ_per_process    set value of bool variable targ_per_process 
  --targ_per_thread     set value of bool variable targ_per_thread 
  --targ_per_pidns      set value of bool variable targ_per_pidns 
  --targ_ms             set value of bool variable targ_ms 
  --targ_tgid           set value of pid_t variable targ_tgid 

Built with eunomia-bpf framework.
See https://github.com/eunomia-bpf/eunomia-bpf for more information.

$ sudo ecli examples/bpftools/runqlat/package.json
key =  4294967295
comm = rcu_preempt

     (unit)              : count    distribution
         0 -> 1          : 9        |****                                    |
         2 -> 3          : 6        |**                                      |
         4 -> 7          : 12       |*****                                   |
         8 -> 15         : 28       |*************                           |
        16 -> 31         : 40       |*******************                     |
        32 -> 63         : 83       |****************************************|
        64 -> 127        : 57       |***************************             |
       128 -> 255        : 19       |*********                               |
       256 -> 511        : 11       |*****                                   |
       512 -> 1023       : 2        |                                        |
      1024 -> 2047       : 2        |                                        |
      2048 -> 4095       : 0        |                                        |
      4096 -> 8191       : 0        |                                        |
      8192 -> 16383      : 0        |                                        |
     16384 -> 32767      : 1        |                                        |

$ sudo ecli examples/bpftools/runqlat/package.json --targ_per_process
key =  3189
comm = cpptools

     (unit)              : count    distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 1        |***                                     |
        16 -> 31         : 2        |*******                                 |
        32 -> 63         : 11       |****************************************|
        64 -> 127        : 8        |*****************************           |
       128 -> 255        : 3        |**********                              |
```

## details in bcc

```text
Demonstrations of runqlat, the Linux eBPF/bcc version.


This program summarizes scheduler run queue latency as a histogram, showing
how long tasks spent waiting their turn to run on-CPU.

Here is a heavily loaded system:

# ./runqlat 
Tracing run queue latency... Hit Ctrl-C to end.
^C
     usecs               : count     distribution
         0 -> 1          : 233      |***********                             |
         2 -> 3          : 742      |************************************    |
         4 -> 7          : 203      |**********                              |
         8 -> 15         : 173      |********                                |
        16 -> 31         : 24       |*                                       |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 30       |*                                       |
       128 -> 255        : 6        |                                        |
       256 -> 511        : 3        |                                        |
       512 -> 1023       : 5        |                                        |
      1024 -> 2047       : 27       |*                                       |
      2048 -> 4095       : 30       |*                                       |
      4096 -> 8191       : 20       |                                        |
      8192 -> 16383      : 29       |*                                       |
     16384 -> 32767      : 809      |****************************************|
     32768 -> 65535      : 64       |***                                     |

The distribution is bimodal, with one mode between 0 and 15 microseconds,
and another between 16 and 65 milliseconds. These modes are visible as the
spikes in the ASCII distribution (which is merely a visual representation
of the "count" column). As an example of reading one line: 809 events fell
into the 16384 to 32767 microsecond range (16 to 32 ms) while tracing.

I would expect the two modes to be due the workload: 16 hot CPU-bound threads,
and many other mostly idle threads doing occasional work. I suspect the mostly
idle threads will run with a higher priority when they wake up, and are
the reason for the low latency mode. The high latency mode will be the
CPU-bound threads. More analysis with this and other tools can confirm.


A -m option can be used to show milliseconds instead, as well as an interval
and a count. For example, showing three x five second summary in milliseconds:

# ./runqlat -m 5 3
Tracing run queue latency... Hit Ctrl-C to end.

     msecs               : count     distribution
         0 -> 1          : 3818     |****************************************|
         2 -> 3          : 39       |                                        |
         4 -> 7          : 39       |                                        |
         8 -> 15         : 62       |                                        |
        16 -> 31         : 2214     |***********************                 |
        32 -> 63         : 226      |**                                      |

     msecs               : count     distribution
         0 -> 1          : 3775     |****************************************|
         2 -> 3          : 52       |                                        |
         4 -> 7          : 37       |                                        |
         8 -> 15         : 65       |                                        |
        16 -> 31         : 2230     |***********************                 |
        32 -> 63         : 212      |**                                      |

     msecs               : count     distribution
         0 -> 1          : 3816     |****************************************|
         2 -> 3          : 49       |                                        |
         4 -> 7          : 40       |                                        |
         8 -> 15         : 53       |                                        |
        16 -> 31         : 2228     |***********************                 |
        32 -> 63         : 221      |**                                      |

This shows a similar distribution across the three summaries.


A -p option can be used to show one PID only, which is filtered in kernel for
efficiency. For example, PID 4505, and one second summaries:

# ./runqlat -mp 4505 1
Tracing run queue latency... Hit Ctrl-C to end.

     msecs               : count     distribution
         0 -> 1          : 1        |*                                       |
         2 -> 3          : 2        |***                                     |
         4 -> 7          : 1        |*                                       |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 25       |****************************************|
        32 -> 63         : 3        |****                                    |

     msecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 2        |**                                      |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 1        |*                                       |
        16 -> 31         : 30       |****************************************|
        32 -> 63         : 1        |*                                       |

     msecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 28       |****************************************|
        32 -> 63         : 2        |**                                      |

     msecs               : count     distribution
         0 -> 1          : 1        |*                                       |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 27       |****************************************|
        32 -> 63         : 4        |*****                                   |
[...]

For comparison, here is pidstat(1) for that process:

# pidstat -p 4505 1
Linux 4.4.0-virtual (bgregg-xxxxxxxx)  02/08/2016  _x86_64_ (8 CPU)

08:56:11 AM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:56:12 AM     0      4505    9.00    3.00    0.00   12.00     0  bash
08:56:13 AM     0      4505    7.00    5.00    0.00   12.00     0  bash
08:56:14 AM     0      4505   10.00    2.00    0.00   12.00     0  bash
08:56:15 AM     0      4505   11.00    2.00    0.00   13.00     0  bash
08:56:16 AM     0      4505    9.00    3.00    0.00   12.00     0  bash
[...]

This is a synthetic workload that is CPU bound. It's only spending 12% on-CPU
each second because of high CPU demand on this server: the remaining time
is spent waiting on a run queue, as visualized by runqlat.


Here is the same system, but when it is CPU idle:

# ./runqlat 5 1
Tracing run queue latency... Hit Ctrl-C to end.

     usecs               : count     distribution
         0 -> 1          : 2250     |********************************        |
         2 -> 3          : 2340     |**********************************      |
         4 -> 7          : 2746     |****************************************|
         8 -> 15         : 418      |******                                  |
        16 -> 31         : 93       |*                                       |
        32 -> 63         : 28       |                                        |
        64 -> 127        : 119      |*                                       |
       128 -> 255        : 9        |                                        |
       256 -> 511        : 4        |                                        |
       512 -> 1023       : 20       |                                        |
      1024 -> 2047       : 22       |                                        |
      2048 -> 4095       : 5        |                                        |
      4096 -> 8191       : 2        |                                        |

Back to a microsecond scale, this time there is little run queue latency past 1
millisecond, as would be expected.


Now 16 threads are performing heavy disk I/O:

# ./runqlat 5 1
Tracing run queue latency... Hit Ctrl-C to end.

     usecs               : count     distribution
         0 -> 1          : 204      |                                        |
         2 -> 3          : 944      |*                                       |
         4 -> 7          : 16315    |*********************                   |
         8 -> 15         : 29897    |****************************************|
        16 -> 31         : 1044     |*                                       |
        32 -> 63         : 23       |                                        |
        64 -> 127        : 128      |                                        |
       128 -> 255        : 24       |                                        |
       256 -> 511        : 5        |                                        |
       512 -> 1023       : 13       |                                        |
      1024 -> 2047       : 15       |                                        |
      2048 -> 4095       : 13       |                                        |
      4096 -> 8191       : 10       |                                        |

The distribution hasn't changed too much. While the disks are 100% busy, there
is still plenty of CPU headroom, and threads still don't spend much time
waiting their turn.


A -P option will print a distribution for each PID:

# ./runqlat -P
Tracing run queue latency... Hit Ctrl-C to end.
^C

pid = 0
     usecs               : count     distribution
         0 -> 1          : 351      |********************************        |
         2 -> 3          : 96       |********                                |
         4 -> 7          : 437      |****************************************|
         8 -> 15         : 12       |*                                       |
        16 -> 31         : 10       |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 16       |*                                       |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 0        |                                        |
      1024 -> 2047       : 0        |                                        |
      2048 -> 4095       : 0        |                                        |
      4096 -> 8191       : 0        |                                        |
      8192 -> 16383      : 1        |                                        |

pid = 12929
     usecs               : count     distribution
         0 -> 1          : 1        |****************************************|
         2 -> 3          : 0        |                                        |
         4 -> 7          : 1        |****************************************|

pid = 12930
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 1        |****************************************|
        32 -> 63         : 0        |                                        |
        64 -> 127        : 1        |****************************************|

pid = 12931
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 1        |********************                    |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 2        |****************************************|

pid = 12932
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 1        |****************************************|
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 1        |****************************************|

pid = 7
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 426      |*************************************   |
         4 -> 7          : 457      |****************************************|
         8 -> 15         : 16       |*                                       |

pid = 9
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 425      |****************************************|
         8 -> 15         : 16       |*                                       |

pid = 11
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 10       |****************************************|

pid = 14
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 8        |****************************************|
         4 -> 7          : 2        |**********                              |

pid = 18
     usecs               : count     distribution
         0 -> 1          : 414      |****************************************|
         2 -> 3          : 0        |                                        |
         4 -> 7          : 20       |*                                       |
         8 -> 15         : 8        |                                        |

pid = 12928
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 1        |****************************************|
         8 -> 15         : 0        |                                        |
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 1        |****************************************|

pid = 1867
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 15       |****************************************|
        16 -> 31         : 1        |**                                      |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 4        |**********                              |

pid = 1871
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 2        |****************************************|
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 1        |********************                    |

pid = 1876
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 1        |****************************************|
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 1        |****************************************|

pid = 1878
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 3        |****************************************|

pid = 1880
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 3        |****************************************|

pid = 9307
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 1        |****************************************|

pid = 1886
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 1        |********************                    |
         8 -> 15         : 2        |****************************************|

pid = 1888
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 3        |****************************************|

pid = 3297
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 1        |****************************************|

pid = 1892
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 1        |********************                    |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 2        |****************************************|

pid = 7024
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 4        |****************************************|

pid = 16468
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 3        |****************************************|

pid = 12922
     usecs               : count     distribution
         0 -> 1          : 1        |****************************************|
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 1        |****************************************|
        16 -> 31         : 1        |****************************************|
        32 -> 63         : 0        |                                        |
        64 -> 127        : 1        |****************************************|

pid = 12923
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 1        |********************                    |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 2        |****************************************|
       128 -> 255        : 0        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 1        |********************                    |
      1024 -> 2047       : 1        |********************                    |

pid = 12924
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 2        |********************                    |
         8 -> 15         : 4        |****************************************|
        16 -> 31         : 1        |**********                              |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 0        |                                        |
      1024 -> 2047       : 1        |**********                              |

pid = 12925
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 1        |****************************************|

pid = 12926
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 1        |****************************************|
         4 -> 7          : 0        |                                        |
         8 -> 15         : 1        |****************************************|
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 1        |****************************************|

pid = 12927
     usecs               : count     distribution
         0 -> 1          : 1        |****************************************|
         2 -> 3          : 0        |                                        |
         4 -> 7          : 1        |****************************************|


A -L option will print a distribution for each TID:

# ./runqlat -L
Tracing run queue latency... Hit Ctrl-C to end.
^C

tid = 0
     usecs               : count     distribution
         0 -> 1          : 593      |****************************            |
         2 -> 3          : 829      |****************************************|
         4 -> 7          : 300      |**************                          |
         8 -> 15         : 321      |***************                         |
        16 -> 31         : 132      |******                                  |
        32 -> 63         : 58       |**                                      |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 13       |                                        |

tid = 7
     usecs               : count     distribution
         0 -> 1          : 8        |********                                |
         2 -> 3          : 19       |********************                    |
         4 -> 7          : 37       |****************************************|
[...]


And a --pidnss option (short for PID namespaces)  will print for each PID
namespace, for analyzing container performance:

# ./runqlat --pidnss -m
Tracing run queue latency... Hit Ctrl-C to end.
^C

pidns = 4026532870
     msecs               : count     distribution
         0 -> 1          : 40       |****************************************|
         2 -> 3          : 1        |*                                       |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 0        |                                        |
        32 -> 63         : 2        |**                                      |
        64 -> 127        : 5        |*****                                   |

pidns = 4026532809
     msecs               : count     distribution
         0 -> 1          : 67       |****************************************|

pidns = 4026532748
     msecs               : count     distribution
         0 -> 1          : 63       |****************************************|

pidns = 4026532687
     msecs               : count     distribution
         0 -> 1          : 7        |****************************************|

pidns = 4026532626
     msecs               : count     distribution
         0 -> 1          : 45       |****************************************|
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 3        |**                                      |

pidns = 4026531836
     msecs               : count     distribution
         0 -> 1          : 314      |****************************************|
         2 -> 3          : 1        |                                        |
         4 -> 7          : 11       |*                                       |
         8 -> 15         : 28       |***                                     |
        16 -> 31         : 137      |*****************                       |
        32 -> 63         : 86       |**********                              |
        64 -> 127        : 1        |                                        |

pidns = 4026532382
     msecs               : count     distribution
         0 -> 1          : 285      |****************************************|
         2 -> 3          : 5        |                                        |
         4 -> 7          : 16       |**                                      |
         8 -> 15         : 9        |*                                       |
        16 -> 31         : 69       |*********                               |
        32 -> 63         : 25       |***                                     |

Many of these distributions have two modes: the second, in this case, is
caused by capping CPU usage via CPU shares.


USAGE message:

# ./runqlat -h
usage: runqlat.py [-h] [-T] [-m] [-P] [--pidnss] [-L] [-p PID]
                  [interval] [count]

Summarize run queue (scheduler) latency as a histogram

positional arguments:
  interval            output interval, in seconds
  count               number of outputs

optional arguments:
  -h, --help          show this help message and exit
  -T, --timestamp     include timestamp on output
  -m, --milliseconds  millisecond histogram
  -P, --pids          print a histogram per process ID
  --pidnss            print a histogram per PID namespace
  -L, --tids          print a histogram per thread ID
  -p PID, --pid PID   trace this PID only

examples:
    ./runqlat            # summarize run queue latency as a histogram
    ./runqlat 1 10       # print 1 second summaries, 10 times
    ./runqlat -mT 1      # 1s summaries, milliseconds, and timestamps
    ./runqlat -P         # show each PID separately
    ./runqlat -p 185     # trace PID 185 only

```
