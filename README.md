### FLAME GRAPH OFF-CPU
Investigate and identify why the application is off-CPU.

## Requirements:

[gcc](https://gcc.gnu.org/) 

[bcc](https://github.com/iovisor/bcc/blob/master/INSTALL.md)

[FlameGraph](https://github.com/brendangregg/FlameGraph)

[App](/blocky.c)

## Compile and run the APP

Get the app
```
$ cd
$ mkdir performance
$ cd performance/
$ wget https://github.com/Jeskz0rd/Off-CPU/blob/master/blocky.c
```
Compile and run 

```
$ gcc -g -fno-omit-frame-pointer -fno-inline -pthread  blocky.c -o blocky
$  ./blocky
[*] Request processor initialized.
[*] Request processor initialized.
[*] Ready to process requests.
[*] Backend handler initialized.
[-] Handled 1000 requests.
[-] Handled 2000 requests.
[-] Handled 3000 requests.
[-] Handled 4000 requests.
[-] Handled 5000 requests.
[-] Handled 6000 requests.
```
The blocky application must be running for the next steps...

![Resources](/img/off-000.png)

Apparentely the application is running using lots of resources... However, when I filtered it at "htop" it doesn't look like the continuous message "Handled thousands requests". Quite insteresting... 

Let's check it a little more.

## Stack traces - ON - OFF

The [cpudist](https://github.com/iovisor/bcc/blob/master/tools/cpudist.py) summarize "on" - and "off" CPU time per task as a histogram.

Bellow the parameters that can be used with cpudist

```
    cpudist              # summarize on-CPU time as a histogram
    cpudist -O           # summarize off-CPU time as a histogram
    cpudist 1 10         # print 1 second summaries, 10 times
    cpudist -mT 1        # 1s summaries, milliseconds, and timestamps
    cpudist -P           # show each PID separately
    cpudist -p 185       # trace PID 185 only
```

First, We check "on-CPU" time.

```
$ cd
$ cd performance/
$ git clone https://github.com/iovisor/bcc
$ cd bcc/tools/
$ ./cpudist.py -p $(pidof blocky)
Tracing on-CPU time... Hit Ctrl-C to end.
^C
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 0        |                                        |
         8 -> 15         : 0        |                                        |
        16 -> 31         : 0        |                                        |
        32 -> 63         : 0        |                                        |
        64 -> 127        : 0        |                                        |
       128 -> 255        : 0        |                                        |
       256 -> 511        : 0        |                                        |
       512 -> 1023       : 0        |                                        |
      1024 -> 2047       : 1        |*                                       |
      2048 -> 4095       : 0        |                                        |
      4096 -> 8191       : 3        |****                                    |
      8192 -> 16383      : 14       |*********************                   |
     16384 -> 32767      : 26       |****************************************|
     32768 -> 65535      : 5        |*******                                 |
     65536 -> 131071     : 4        |******                                  |
    131072 -> 262143     : 4        |******                                  |
    262144 -> 524287     : 1        |*                                       |
```

And now, let's compare with off-CPU time

```
$ ./cpudist.py -O -p $(pidof blocky)
Tracing off-CPU time... Hit Ctrl-C to end.
^C
     usecs               : count     distribution
         0 -> 1          : 0        |                                        |
         2 -> 3          : 0        |                                        |
         4 -> 7          : 13       |                                        |
         8 -> 15         : 38       |                                        |
        16 -> 31         : 35       |                                        |
        32 -> 63         : 8        |                                        |
        64 -> 127        : 12       |                                        |
       128 -> 255        : 420      |                                        |
       256 -> 511        : 28536    |****************************************|
       512 -> 1023       : 3610     |*****                                   |
      1024 -> 2047       : 194      |                                        |
      2048 -> 4095       : 117      |                                        |
      4096 -> 8191       : 89       |                                        |
      8192 -> 16383      : 8706     |************                            |
     16384 -> 32767      : 8516     |***********                             |
     32768 -> 65535      : 2        |                                        |
```

## Flame Graph

To understand it better, let`s get Flame Graph.

```
$ cd ~/performance/
$ git clone https://github.com/brendangregg/FlameGraph
```

Parameters that can be used with profile tool, a source to Flame Graph

```
    ./profile             # profile stack traces at 49 Hertz until Ctrl-C
    ./profile -F 99       # profile stack traces at 99 Hertz
    ./profile -c 1000000  # profile stack traces every 1 in a million events
    ./profile 5           # profile at 49 Hertz for 5 seconds only
    ./profile -f 5        # output in folded format for flame graphs
    ./profile -p 185      # only profile threads for PID 185
    ./profile -U          # only show user space stacks (no kernel)
    ./profile -K          # only show kernel space stacks (no user)
```

Let's profile the CPU...
After few seconds hit CTRL + C

```
$ cd ~/performance/bcc/tools/
$ ./profile.py -F 99 -f -p $(pidof blocky) > ~/performance/FlameGraph/trace_blocky_profile
^C
```
Now we have the sample, it is possible to create the Flame Graph

```
$ cd ~/performance/FlameGraph/
$ ./flamegraph.pl trace_blocky_profile > blocky_profile.svg
```

As it is possible to see bellow, "request_processor" and "do_work" are consuming lots of CPU.

![Flame Graph - Profile ](/img/blocky_profile.svg)

Well, let's go deeper to identify the OFF-CPU. This step, "offcputime" tool will be used.

Bellow the parameters that can be used with offcputime

```
    ./offcputime             # trace off-CPU stack time until Ctrl-C
    ./offcputime 5           # trace for 5 seconds only
    ./offcputime -f 5        # 5 seconds, and output in folded format
    ./offcputime -m 1000     # trace only events that last more than 1000 usec
    ./offcputime -M 10000    # trace only events that last less than 10000 usec
    ./offcputime -p 185      # only trace threads for PID 185
    ./offcputime -t 188      # only trace thread 188
    ./offcputime -u          # only trace user threads (no kernel)
    ./offcputime -k          # only trace kernel threads (no user)
    ./offcputime -U          # only show user space stacks (no kernel)
    ./offcputime -K          # only show kernel space stacks (no user)
```

Create the sample using the following, after few seconds hit CTRL+C

```
$ cd ~/performance/bcc/tools/
$ ./offcputime.py -f -p $(pidof blocky) > ~/performance/FlameGraph/trace_blocky_offcpu
$ cd ~/performance/FlameGraph/
$ ./flamegraph.pl trace_blocky_offcpu > blocky_offcpu.svg
```

![Flame Graph - Off-CPU](/img/blocky_offcpu.svg)

As it is possible to see, "backend_handler" calls nanosleep and "request_processor" calls "lock wait". 

Checking the "blocky.c" source code, it is pretty easy to identify now that "backend_handler" is the offensor, which gets locked and sleep (10ms) while "request_processor" needs it to go on but get stuck on it!
