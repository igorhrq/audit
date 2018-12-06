# audit
Measure and Troubleshoot Linux Memory Resource Usage

Tracking down physical memory usage (RAM + swap) and identifying amount of memory needed for current workload on a Debian server. 

Install Performance Monitoring Tools
Install sysstat:

# apt-get update && apt-get install sysstat
For RHEL/CentOS, do the following:

# yum install -y sysstat
The sysstat package contains the sar system performance tool which we’re going to use today.

Make sure that sar is enabled in /etc/default/sysstat. If not enabled, do it.

You may also want to change the history value in /etc/sysstat/sysstat to something different than 7 days:

HISTORY=60
Note that if value is greater than 28, then log files will be kept in multiple directories, one for each month.

By default sysstat will collect data every 10 minutes. You can change this by modifying the cronjob /etc/cron.d/sysstat.

Finally, restart the service:

# service sysstat restart
Measure Memory Usage
Memory Usage with free
The free command displays the total amount of free and used physical and swap memory in the system, as well as the buffers used by the kernel.

Show memory usage report in megabytes (-m):

$ free -m
             total   used  free  shared  buffers  cached
Mem:           998   985     12       0       25     615
-/+ buffers/cache:   344    654
Swap:          967   104    863
Reported values have the following meanings:

998 – total amount of physical memory installed (all values are in MB).
985 – amount of memory that is currently in use from the OS perspective. This does include buffers and cached.
12 – amount of memory that is not in use in any way from the OS perspective.
0 – shared memory, obsolete.
25 – amount of memory that is buffered. In simple words, buffers are used for caching of filesystem metadata (permissions, location, etc.) and tracking in-flight pages.
615 – amount of memory that is cached. Cache contains data that has already been read from the disk and is kept in memory for possible future use, f.e a pdf file or web browser pages.
344 – amount of memory that is currently in use from application’s perspective. This does not include buffers and cached.
654 – amount of memory that is free from application’s perspective. This does include buffers and cached.
967 – total amount of swap memory.
104 – amount of swap memory that is in use.
863 – amount of swap memory that is not in use.
344 MB is the amount of memory that is actually in use by running processes. 654 MB is the amount of memory to be considered “free on request”, as it can be easily freed. The sum of both is the total amount of physical memory: 344 MB + 654 MB = 998 MB.

Note well that any unused RAM is a waste of money. Linux system always tries to use all available physical memory for buffers and cache to run faster. Buffers and cache help to reduce I/O disk operations.

Memory Usage with /proc/meminfo
Another place to look for memory usage statistics is /proc/meminfo:

$ cat /proc/meminfo | head
MemTotal:        1022744 kB
MemFree:           14700 kB
Buffers:            4532 kB
Cached:           657328 kB
SwapCached:        10032 kB
Active:           396132 kB
Inactive:         561848 kB
Active(anon):      69940 kB
Inactive(anon):   238392 kB
Active(file):     326192 kB

As with the free command earlier, the output presented by /proc/meminfo may be confusing at first. Pay particular attention to the active memory line (marked in blue). This is the actual amount of memory that is in use by running processes. MemFree field reports the amount of memory that is free from the OS perspective.

Memory Usage with vmstat
The vmstat command reports information about several different resource activities, including memory, CPU, processes, paging, block IO and disks activity. The first reported line gives averages since the last reboot. Default output shows memory usage in KB (1024B).

Display three reports at one second intervals:

$ vmstat 1 3
procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
 0  0 105596  19048  27664 532596    0    0    32     7   52   48  2  1 96  1
 0  0 105596  19048  27680 532620    0    0     0    40  293  484  2  0 97  1
 0  0 105596  19080  27680 532620    0    0     0     0  161  242  1  0 99  0
Display three reports with active (-a) and inactive memory at one second intervals:

$ vmstat -a 1 3
procs -----------memory---------- ---swap-- -----io---- -system-- ----cpu----
 r  b   swpd   free  inact active   si   so    bi    bo   in   cs us sy id wa
 0  0 105596  19100 595012 363880    0    0    32     7   52   48  2  1 96  1
 0  0 105596  19092 595016 363896    0    0     0    20  212  346  1  1 98  0
 0  0 105596  19092 595020 363908    0    0     0     0  151  282  1  0 99  0
Memory activity is marked in blue. The following values are displayed (as per man page):

swpd: the amount of virtual memory used.
free: the amount of idle memory.
buff: the amount of memory used as buffers.
cache: the amount of memory used as cache.
inact: the amount of inactive memory (-a option).
active: the amount of active memory (-a option).
Swap activity is marked in maroon. The following values are displayed (as per man page):

si: Amount of memory swapped in from disk (/s).
so: Amount of memory swapped to disk (/s).
Memory Usage with top
As was mentioned earlier, the top program provides a dynamic real-time view of a running system. Top is a great tool to find out which users and their processes use the most memory at the time of monitoring. To sort view by memory, use a “Ctrl”+”M” combination.

As we may see below (in red), MySQL utilises almost 18% of all available RAM where Nessus takes further 10%.

$ top
top - 21:40:02 up 4 days,  2:32,  1 user,  load average: 0.01, 0.05, 0.05
Tasks: 122 total,   1 running, 120 sleeping,   0 stopped,   1 zombie
%Cpu(s):  0.7 us,  0.8 sy,  0.0 ni, 98.4 id,  0.1 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem:   1022744 total,  1003528 used,    19216 free,    54352 buffers
KiB Swap:   991228 total,      876 used,   990352 free,   507736 cached

  PID USER      PR  NI  VIRT  RES  SHR S  %CPU %MEM    TIME+  COMMAND             
 2860 mysql     20   0  321m 176m 2712 S   1.0 17.7  74:07.10 mysqld
 1923 root      20   0  295m 103m 1476 S   0.4 10.4  89:00.31 nessusd
 2265 www-data  20   0 47264  12m 1976 S   0.0  1.3   0:03.46 apache2
 2254 www-data  20   0 47528  11m  628 S   0.0  1.2   0:02.90 apache2
 3174 www-data  20   0 29384  11m  960 S   0.0  1.1   0:02.69 zmfilter.pl
 3271 www-data  20   0 45948  10m  612 S   0.0  1.0   0:01.74 apache2
 3103 zabbix    20   0 59872   9m 9180 S   0.0  1.0   4:26.57 zabbix_server
 3102 zabbix    20   0 59872 9.9m 9156 S   0.0  1.0   4:28.83 zabbix_server
 3100 zabbix    20   0 59872 9.9m 9140 S   0.0  1.0   4:30.39 zabbix_server
Line 4, marked in blue, shows memory usage in kilobytes since the last refresh. Line 5, marked in indigo, displays swap usage in kilobytes since the last refresh.

Other memory related values that are displayed are:

VIRT: virtual memory size in kilobytes, the total amount of virtual memory used by the task.
RES: resident memory size in kilobytes, the non-swapped physical memory a task has used.
SHR: shared memory size in kilobytes, the amount of shared memory available to a task.
%MEM: memory usage (RES), a task’s currently used share of available physical memory. It’s RES value expressed as percentage.
Memory Usage with ps
The ps command can be used to display information about processes that use the most memory resources. Compared with top, which gives a dynamic real-time view of system resources, ps reports a snapshot of the currently running processes.

Get a snapshot of the 9 most memory consuming processes:

$ ps -eo pid,user,s,comm,size,vsize,rss --sort -size | head
  PID USER     S COMMAND          SIZE    VSZ   RSS
 3059 mysql    S mysqld          330000 343552 172724
 2041 root     S nessusd         326112 333348 114108
 2297 bind     S named           42920  52976  1440
11763 www-data S apache2         26324 108256 27168
 2067 root     S rsyslogd        25452  28208  1092
 3547 www-data S zmfilter.pl     18900  29360  2348
11770 www-data S apache2         16676  98608 18132
11769 www-data S apache2         16380  98312 12880
11768 www-data S apache2         16128  98060 13284
Parameters used are as below:

-e: select all processes.
-o: specify user-defined format.
pid: process ID.
user: user name.
s: minimal state display (one character).
S for sleeping (idle).
R for running.
D for disk sleep (uninterruptible).
Z for zombie (waiting for parent to read it’s exit status).
T for traced or suspended (e.g by SIGTSTP).
W for paging.
comm: command name (only the executable name).
size: memory size in kilobytes.
vsize: total VM size in kilobytes.
rss: resident set size, the non-swapped physical memory that a task has used .
–sort -size: sort size in descending numerical order.
Memory Usage with sar
As was also mentioned earlier in one of my previous post, the sar command gives the report of selected resource activity counters in the system. Sar can display a real-time usage as well as extract historical data.

Display three real-time memory (-r) utilisation reports at one second intervals:

$ sar -r 1 3
Linux 3.2.0-4-686-pae (flames) 	22/02/14 	_i686_	(2 CPU)

21:44:31 kbmemfr kbmemused %memused kbbuffers kbcached kbcommit %commit kbactive kbinact
21:44:32   16660   1006084    98.37      9504   548592  1388856   68.96   395700  561884
21:44:33   16536   1006208    98.38      9504   548612  1388856   68.96   395700  561872
21:44:34   16536   1006208    98.38      9504   548612  1388856   68.96   395700  561872
Average:   16577   1006167    98.38      9504   548605  1388856   68.96   395700  561876
The following values are displayed (as per man page):

kbmemfree: amount of free memory available in kilobytes.
kbmemused: amount of used memory in kilobytes. This does not take into account memory used by the kernel itself.
%memused: percentage of used memory.
kbbuffers: amount of memory used as buffers by the kernel in kilobytes.
kbcached: amount of memory used to cache data by the kernel in kilobytes.
kbcommit: amount of memory in kilobytes needed for current workload. This is an estimate of how much RAM/swap is needed to guarantee that there never is out of memory.
%commit: percentage of memory needed for current workload in relation to the total amount of memory (RAM+swap). This number may be greater than 100% because the kernel usually overcommits memory.
kbactive: amount of active memory in kilobytes (memory that has been used more recently and usually not reclaimed unless absolutely necessary).
kbinact: amount of inactive memory in kilobytes (memory which has been less recently used. It is more eligible to be reclaimed for other purposes).
The interesting fields to pay attention at are “kbcommit” (red) and “%commit” (purple). We have 1GB of physical memory installed plus 1 GB of swap on the server we are running these tests on. This makes the total amount of physical memory to 2GB.  Worth clarifying that loading from swap disk, even an SSD based, is thousands of times slower than loading from RAM!

The kbcommit column, marked in red, indicates that there are 1,37 GB of memory needed to maintain current workload. This is 68% of the total memory available (RAM + swap).

Let’s do a simple test, let’s switch swap off – reduce total physical memory to 1GB – and run the sar command again.

# swapoff -a
$ sar -r 1 3
Linux 3.2.0-4-686-pae (flames) 	22/02/14 	_i686_	(2 CPU)

21:45:08 kbmemfr kbmemused %memused kbbuffers kbcached kbcommit %commit kbactive kbinact
21:45:09   15720   1007024    98.46     10064   549060  1388856  135.80   395848  562740
21:45:10   15720   1007024    98.46     10064   549060  1388856  135.80   395848  562740
21:45:11   15720   1007024    98.46     10064   549060  1388856  135.80   395848  562740
Average:   15720   1007024    98.46     10064   549060  1388856  135.80   395848  562740
What we see now is that the percentage of memory needed for current workload is greater than 100%. This is an expected result as we reduced total physical memory (RAM + swap) by half.

It the percentage is much greater than 100%, it may indicate a memory shortage.

Swap Space Usage with sar
Display three real-time swap space (-S) utilisation reports at one second intervals:

$ sar -S 1 3
Linux 3.2.0-4-686-pae (flames) 	22/02/14 	_i686_	(2 CPU)

21:20:51  kbswpfree  kbswpused  %swpused  kbswpcad  %swpcad
21:20:52  885632        105596     10.65     10032     9.50
21:20:53  885632        105596     10.65     10032     9.50
21:20:54  885632        105596     10.65     10032     9.50
Average:  885632        105596     10.65     10032     9.50
The following values are displayed (as per man page):

kbswpfree: amount of free swap space in kilobytes.
kbswpused: amount of used swap space in kilobytes.
%swpused: percentage of used swap space.
kbswpcad: amount of cached swap memory in kilobytes. This is memory that once was swapped out, is swapped back in but still also is in the swap area.
%swpcad: percentage of cached swap memory in relation to the amount of used swap space.
Historical Memory, Swap and Swapping Statistics with sar
Extract historical memory (-r), swap space (-S) and swapping (-W) statistics records starting (-s) 1 PM and ending (-e) 2 PM time interval:

$ sar -rSW -s 13:00:00 -e 14:00:00
Linux 3.2.0-4-686-pae (flames) 	22/02/14 	_i686_	(2 CPU)

13:05:01  pswpin/s pswpout/s
13:15:01      0.35      0.00
13:25:01      0.00      0.00
13:35:01      0.00      0.00
13:45:01      0.00      0.00
13:55:01      0.00      0.00
Average:      0.07      0.00

13:05:01 kbmemfr kbmemused %memused kbbuffers kbcached kbcommit %commit kbactive kbinact
13:15:01   72604    950140    92.90    119752   485124  1360700   67.56   380456  517556
13:25:01   71140    951604    93.04    120936   485596  1360700   67.56   383268  516392
13:35:01   69412    953332    93.21    122288   485840  1360700   67.56   385148  516124
13:45:01   67964    954780    93.35    123444   486256  1360700   67.56   387216  515620
13:55:01   66320    956424    93.52    124596   486580  1360700   67.56   389328  514988
Average:   69488    953256    93.21    122203   485879  1360700   67.56   385083  516136

13:05:01 kbswpfree kbswpused %swpused  kbswpcad  %swpcad
13:15:01    885548    105680    10.66      9192     8.70
13:25:01    885548    105680    10.66      9192     8.70
13:35:01    885548    105680    10.66      9192     8.70
13:45:01    885548    105680    10.66      9192     8.70
13:55:01    885548    105680    10.66      9192     8.70
Average:    885548    105680    10.66      9192     8.70
The following swapping statistics values are displayed:

pswpin/s: total number of swap pages the system brought in per second.
pswpout/s: total number of swap pages the system brought out per second.
Having historical statistics, these can be compared with real time usage reports to help to identify memory problems and system slowdowns.

Swap Space Usage with smem
Smem is a tool to report physical memory usage. Install it:

# apt-get install smem
For RHEL/CentOS, install from EPEL repository:

# yum install -y smem
Show reversed sorted (-rs) totals (-t):

# smem -t -rs swap | head -n6
  PID User     Command                         Swap      USS      PSS      RSS 
19811 sandy    /usr/bin/dbus-daemon --fork   101372     1404     1550     1972 
22119 root     ntop -cd -i eth0 -u ntop -W    45420   163264   165056   168328 
30636 sandy    /usr/bin/java -ea -Xmx512m     42296   269296   271184   279456 
 9517 sandy    /usr/lib/chromium/chromium     16508   157552   161675   169548 
21068 root     /usr/lib/virtualbox/Virtual    12036  2193916  2198383  2204564
Show reversed sorted (-rs) totals (-t) as a percentage (-p):

# smem -t -p -rs swap | head -n6
  PID User     Command                         Swap      USS      PSS      RSS 
19811 sandy    /usr/bin/dbus-daemon --fork    1.25%    0.02%    0.02%    0.02% 
22119 root     ntop -cd -i eth0 -u ntop -W    0.56%    2.01%    2.03%    2.07% 
30636 sandy    /usr/bin/java -ea -Xmx512m     0.52%    3.33%    3.35%    3.45% 
 9517 sandy    /usr/lib/chromium/chromium     0.20%    1.94%    1.99%    2.09% 
21068 root     /usr/lib/virtualbox/Virtual    0.15%   26.99%   27.05%   27.12%
From the smem man page, unshared memory is reported as the USS (Unique Set Size). The unshared memory (USS) plus a process’s proportion of shared memory is reported as the PSS (Proportional Set Size). The USS and PSS only include physical memory usage. They do not include memory that has been swapped out to disk. RRS (Resident Set Size) is a non-swapped physical memory that a task has used.
