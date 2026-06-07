# 学习笔记
## CHANGELOG.adoc
基本一个月一个版本

## 20260607
### 通信的3种方式测试
1. pub， sub， aeronstat
pub 通过内嵌driver启动，监控/tmp/aeron-basic目录
aeronstat监控/tmp/aeron-basic目录
sub 不启动driver，直接连接到/tmp/aeron-basic目录
通过内存映射直接通信，也可以实时监控消费数据变化

虽然也有channel，但不走udp，直接共享内存了，相当于md是公用的，需要证明（todo）

2. pub， sub
pub 内嵌driver启动，目录随机
sub 内嵌driver启动，目录随机
channel指定udp
所以两者虽然目录不同，但会通过udp通信
