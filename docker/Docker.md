#### Cgroup

全称control group，属于Linux内核提供的一个特性，用于限制和隔离一组进程对系统资源的使用。Cgroup实现了一个通用的进程分组的框架，而不同资源的管理则由各个Cgroup子系统实现。Cgroup实现的子系统及其作用：

| sub system | desc                           |
| ---------- | ------------------------------ |
| devices    | 设备权限控制                         |
| cpuset     | 分配指定的CPU和内存节点                  |
| cpu        | 控制CPU占用率                       |
| cpuacct    | 统计CPU使用情况                      |
| memory     | 限制内存的使用上限                      |
| freezer    | 冻结Cgroup中的进程                   |
| net_cls    | 配合tc（traffic controller）限制网络带宽 |
| net_prio   | 设置进程的网络流量优先级                   |
| huge_tlb   | 限制HugeTLB的使用                   |
| perf_event | 允许Perf工具基于Cgroup分组做性能监测        |

Cgroup的原生接口通过cgroupfs提供，一种虚拟文件系统。

如何使用Cgroup

* 挂载cgroupfs，这个动作一般在启动时已经由linux发行版做好了，可以挂载在任意一个目录上，标准的挂载点是/sys/fs/cgroup

  ```shell
  # mount -t cgroup -o cpuset cpuset /sys/fs/cgroup/cpuset
  ```

* 查看cgroupfs

  ```shell
  # ls /sys/fs/cgroup/cupset
  ```

* 创建Cgroup，通过mkdir创建一个新目录也就创建了一个新的Cgroup

  ```shell
  # mkdir /sys/fs/cgroup/spuset/child
  ```

* 配置Cgroup

  ```shell
  # echo 0 > /sys/fs/cgroup/cpuset/child/cpuset.cpus
  # echo 0 > /sys/fs/cgroup/cpuset/child/cpuset.mems
  ```

  通过以上命令配置这个Cgroup的资源配额，限制这个Cgroup的进程只能在0号CPU 上运行，并且只会从0号内存节点分配内存

* 使用Cgroup

  ```shell
  # echo $$ > /sys/fs/cgroup/cpuset/child/tasks
  ```

  $$ 表示当前进程。以上命令将进程id写入tasks文件中，就可以把这个进程移动到这个Cgroup中，并且，这个进程产生的所有子进程也都会自动放在这个Cgroup中，这是Cgroup才真正起作用。


Cgroup子系统

* cpuset子系统

