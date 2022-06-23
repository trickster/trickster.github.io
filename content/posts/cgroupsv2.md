+++
title = "cgroupsv2 on Linux"
date = 2022-06-23T10:45:55+09:00
type = "post"
description = "cgroupsv2 on Fedora 35"
in_search_index = true
[taxonomies]
tags = ["Linux", "Tools"]
+++

## Check the kernel params

```sh
cat /etc/default/grub
> GRUB_CMDLINE_LINUX=".... systemd_unified_cgroup_heirarchy=1"
```

You can change the above with

```sh
sudo grubby --update-kernel=ALL --args="systemd.unified_cgroup_hierarchy=1
```

### One last check

```sh
grep -c cgroup /proc/mounts # Count cgroup mounts
```

If the above shows a value > 1, then you need to reboot

## Filesystem

- On boot v2 heirarchy is at `/sys/fs/cgroup`.

```sh
[root ~]$ ls /sys/fs/cgroup/
cgroup.controllers      cpuset.cpus.effective  io.pressure                    sys-kernel-config.mount
cgroup.max.depth        cpuset.mems.effective  io.prio.class                  sys-kernel-debug.mount
cgroup.max.descendants  cpu.stat               io.stat                        sys-kernel-tracing.mount
cgroup.procs            dev-hugepages.mount    memory.numa_stat               system.slice
cgroup.stat             dev-mqueue.mount       memory.pressure                user.slice
cgroup.subtree_control  init.scope             memory.stat
cgroup.threads          io.cost.model          misc.capacity
cpu.pressure            io.cost.qos            sys-fs-fuse-connections.mount
```

## Cgroups

- Mechanism for heirarchically grouping processes
- set of controllers (kernel components) that manage, control and monitor processes in cgroups
- interface via psuedo filesystem
- manipulated manually via shell commands, programs, systemd, docker etc.

### Uses

- limit %CPU, memory, set priorities for resource allocation and monitoring

## What are cgroups

- group of processes that are bound together for resource management (one kind of namespaces in linux kernel)
- arranged in heirarchy
- can have zero or more child cgroups
- inherit control settings from parent

## FS interface

- `mkdir` / `rmdir` will automatically populate/delete the directories

## PID controller example

List of processes running in `mygrp` cgroup

```sh
$ pwd
/sys/fs/cgroup
$ mkdir mygrp
$ cd mygrp/
$ cat cgroup.
cgroup.controllers      cgroup.max.depth        cgroup.max.descendants  cgroup.procs            cgroup.stat             cgroup.subtree_control  cgroup.threads

$ cat cgroup.subtree_control
memory pids
$ echo $$ > mygrp/cgroup.procs # This shell is in that cgroup
$ cat /proc/$$/cgroup
0::/mygrp

$ cat mygrp/cgroup.procs  #check cgroup membership for this process
8068
8099

$ # cat has 8068 pid, because this process is starting from this shell and this shell pid is in mygrp cgroup

$ cat mygrp/pids.current
2
$ cat mygrp/pids.max
max
$ echo 5 > mygrp/pids.max
$ for j in $(seq 1 5); do sleep 60 & done
[1] 8108
[2] 8109
[3] 8110
[4] 8111
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: Resource temporarily unavailable

$ cat pids.current
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: Resource temporarily unavailable

$ cat pids.current
[1]   Done                    sleep 60
[2]   Done                    sleep 60
[3]-  Done                    sleep 60
[4]+  Done                    sleep 60

$ for j in $(seq 1 5); do sleep 60 & done
[1] 8158
[2] 8159
[3] 8160
[4] 8161
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
bash: fork: retry: Resource temporarily unavailable
^Cbash: fork: Interrupted system call

$ echo $$
8068

# Cleaning up resources
$ rmdir mygrp
rmdir: failed to remove 'mygrp': Device or resource busy

# put the current shell in the root cgroup
$ echo $$ > /sys/fs/cgroup/cgroup.procs

$ rmdir mygrp

```

### List of controllers available

- cpu
  - bandwidth strictly limits cpu
- cpuset
  - control CPU and memory affinity
    - pin cgroup to one cpu/subset of cpus or memory nodes
    - dynamically manage placement of app components
    - advanced usage
- memory
  - soft limits & hard limits
- io
  - same as cpu
- devices
  - important for containers
  - limit devices
  - example: inside container, disallow access to devices other than /dev/null/zero/full
  - **control is done by attaching ebpf program to cgroup**
- pids
  - limit number of pids
- freezer
  - freeze and thaw group of processes
  - used for container migration, checkpoint/restore
- perf-event
  - cgroup perf monitoring
- rdma
  - control rdma resources
- hugetlb
  - limit usage of huge pages per cgroup

### Enable/disable cgroup controllers

cgroup.controllers (available) & cgroup.subtree_control (enabled). Some implicit controllers are always available, like freezer, perf_event

```sh
$ cat cgroup.controllers
cpuset cpu io memory hugetlb pids misc

$ cat cgroup.subtree_control
memory pids
$ cat /sys/fs/cgroup/mygrp/pids.current
5
$ cat /sys/fs/cgroup/mygrp/pids.current
1
```

> Read Linux programming interface book

## CPU example

```sh
$ mkdir grp1
$ ls grp1/cpu.*
grp1/cpu.pressure  grp1/cpu.stat
$ # some cpu controls are enabled in grp1 because root cgroup has no such enabled

$ echo '+pids' > cgroup.subtree_control
$ echo '+cpu' > cgroup.subtree_control
$ cat cgroup.subtree_control
cpu memory pids
$ ls grp1/cpu.*
grp1/cpu.idle  grp1/cpu.max.burst  grp1/cpu.stat    grp1/cpu.weight.nice
grp1/cpu.max   grp1/cpu.pressure   grp1/cpu.weight

$ # now I got those controls in grp1 level


$ ls grp1/cpu.*
grp1/cpu.pressure  grp1/cpu.stat
$ echo '+cpu' > cgroup.subtree_control

$ ls grp1/cpu.*
grp1/cpu.idle  grp1/cpu.max.burst  grp1/cpu.stat    grp1/cpu.weight.nice
grp1/cpu.max   grp1/cpu.pressure   grp1/cpu.weight

$ echo '20000 100000' > grp1/cpu.max

$cat grp1/cpu.max
20000 100000

$ echo 23148 > grp1/cgroup.procs
```

CPU dropped in usage (20% as per fraction)

```sh
[23148]  %CPU = 99.57; totCPU = 77.000
[23148]  %CPU = 99.59; totCPU = 78.000
[23148]  %CPU = 27.48; totCPU = 79.000
[23148]  %CPU = 20.00; totCPU = 80.000
[23148]  %CPU = 20.00; totCPU = 81.000
[23148]  %CPU = 20.00; totCPU = 82.000
```

`grp1` controls that are enabled are `root/cgroup.subtree_control`

```sh
[root@siva-fedbox cgroup] cat grp1/cgroup.controllers
cpu memory pids
[root@siva-fedbox cgroup] cat cgroup.subtree_control
cpu memory pids
```

## advanced usage

- instead each cgroup there is `cgroup.events` containing k-v pairs

```sh
$ cat grp1/cgroup.events
populated 0 # 1 == subheirarchy contains live processes, 0 == no processes
frozen 0 
```

- can monitor cgroup.events file, to get notification of changes of keys
  - inotify - generate IN_MODIFY
  - open fd, and monitor using select, epoll, poll
  - after notification, parse cgroup.events to find populated key

- one process can monitor mutiple cgroup.events file (not possible in cgroupv1)

```sh
$ mkdir newcgrp
$ sleep 10000 &
[1] 23614

$ echo 23614 >> newcgrp/cgroup.procs

# Result
$ while inotifywait -q -e modify newcgrp/cgroup.events ; do grep populated newcgrp/cgroup.events ; done

newcgrp/cgroup.events MODIFY
populated 1 # atleast one process

$ kill -9 23614

newcgrp/cgroup.events MODIFY
populated 0
```

## Ownership

More [here](https://man7.org/conf/ndctechtown2021/cgroups-v2-part-2-diving-deeper-NDC-TechTown-2021-Kerrisk.pdf)

- delegater - priveleged user who owns parent cgroup
- delegatee - less privileged user who is assigned management of a subhierarchy under parent cgroup
- delegater grants delegatee write access to certain files
- change ownership of dir what will be root of delegated subtree
  - cgroups.proc
  - cgroup.subtree_control
  - any other filename listed in /sys/kernel/cgroup/delegate

## Creating a container

- Create a cgroup using `mkdir` or `sudo cgcreate -g cpu,cpuset,memory,io,pids:siva`, this creates all resources (everything is unified). Set whatever limits you want using `cgset` or following the kernel [doc][2] or [this][4]

- Once cgroup is created, you can use `unshare` to create namespaces

```sh
sudo cgexec -g "cpu,memory::/siva" \
    unshare -fmuipn --mount-proc \
    chroot "$PWD" \
    /bin/sh -c "
        /bin/mount -t proc proc /proc &&
        hostname mycontainer &&
        /usr/bin/fish"
```

- Delete cgroups using `sudo cgdelete -g cpu:/siva`

## Crashing the container

```sh
sudo cgset -r memory.max=300000000 siva # Set the maximum memory limit for cgroup siva
wget bit.ly/fish-container -O fish.tar # download fish image
mkdir myroot; cd myroot
tar -xf ../fish.tar
sudo cgcreate -g cpu,cpuset,memory,io,pids:siva # create a cgroup
sudo cgset -r memory.swap.max=0 siva # set swap to 0
sudo cgset -r memory.max=300000000 siva # set memory max to 300MB

# execute the command in that cgroup, `unshare` for creating new namespaces
# chroot for making the current directory root, mount proc, change hostname

sudo cgexec -g "cpu,memory::/siva" \
    unshare -fmuipn --mount-proc \
    chroot "$PWD" \
    /bin/sh -c "
        /bin/mount -t proc proc /proc &&
        hostname mycontainer &&
        /usr/bin/fish" 

```

This creates 3 pids, you can check it in `/sys/fs/cgroup/siva/pids.current`

```sh
root        2482  0.0  0.0   5580   912 pts/0    S    03:55   0:00 unshare -fmuipn --mount-proc chroot ....
root        2483  0.0  0.0   1520     4 pts/0    S    03:55   0:00 /bin/sh -c  /bin/mount -t proc proc /pro
root        2486  0.3  0.0  10468  3672 pts/0    S+   03:55   0:00 /usr/bin/fish
```

In your new namespace/container, list of processes are

```sh
    1 root       0:00 /bin/sh -c /bin/mount -t proc proc /proc && hostname mycontainer && ...
    4 root       0:00 /usr/bin/fish
   31 root       0:00 ps
```

You can check the current usage with `sudo cgget -r memory.current siva`. We crash the container by allocating >300MB of memory, and checking memory usage simultaneously.

In your fish shell,

```text
root@mycontainer /# set A 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
root@mycontainer /# for a in (seq 10)
                        set A = "$A$A"
                    end
root@mycontainer /# for a in (seq 10)
                        set A = "$A$A"
                    end
Killed
```

## Support

- `cgroupv1` tools like `cgcreate` and `cgset` etc. (from libcgroup) has [support][3] for `cgroupsv2`. These tools have the same interface as `cgroupv1`, that's why it doesn't feel natural

## Resources

1. [Redhat docs](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/8/html/managing_monitoring_and_updating_the_kernel/using-cgroups-v2-to-control-distribution-of-cpu-time-for-applications_managing-monitoring-and-updating-the-kernel)
2. [Cgroups v2](https://man7.org/conf/ndctechtown2021/cgroups-v2-part-1-intro-NDC-TechTown-2021-Kerrisk.pdf)

[2]: https://www.kernel.org/doc/html/latest/admin-guide/cgroup-v2.html
[3]: https://github.com/libcgroup/libcgroup/issues/12
[4]: https://facebookmicrosites.github.io/cgroup2/docs/memory-controller.html
