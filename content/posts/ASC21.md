---
title: "ASC21丢人日记-Part1"
date: 2021-05-12T22:15:36+08:00
draft: false
tags: ["ASC"]
summary: ASC22不丢人
---

### 初赛

由于疫情的原因,去年的ASC没了,于是就把比赛顺延到了今年. 我们的队长依然是跳跳鱼, 队伍成员基本没有太大的变化, 除了我们负责LE那道题的队友忙于paper, 于是换成了rush负责LE, 娜娜也放弃QuEST转到了LE那道题(虽然我们LE也凉了, 不过那是后话). 当时甚至没有A100, 所以我们所有的比赛方案包括Benchmark都是针对V100, 所以在HPL和HPCG那里ampare架构给我带来了一点小小的麻烦.

第一年本来我也是搞QuEST, 虽然我大部分时间都在搞Benchmark和Cluster Archetecture的事情(为什么我要浪费时间搞这些事情, 建议参考曾经我们某个队员的光荣事迹, 建议那个人自裁). 总之自从宣布比赛延期之后, 我们队就陷入了长时间的摸鱼中. 去年年底发布了PRESTO这个赛题, 我反正没有事情, 于是我就去做那道题目了, 就这样QuEST就剩下二极管带小朋友在做了, 当然我们队新来的王则可老师非常懂这个, 于是其实当时来看QeEST挺稳的. 

我们QuEST的后端当时做了线路优化, 后来王老师来了之后让二极管加上了GPU和多节点多GPU的优化, 直接让QFT和Random两个点的速度提升到(我忘了)秒. 我们初赛的LE跑的其实是Bert的Baseline, 这个Baseline其实效果还可以, 反正别的队伍初赛也就是这点分数, 差距不大. 我们初赛的PRESTO也基本上是我做的, 虽然我有点忙(找工作)的那几天把部分工作让跳跳鱼帮我写一下, 后来我改了一下代码, 加了个消息队列处理了一下输出的问题Pipeline这边就差不多长这样了, 其他主要是做了一下CPU的Profiling, 看了一下HostSpot, 发现基本上都在一些奇怪的地方浪费时间, 最后简单手动unroll了循环, 修改了某些令人窒息的诡异操作(比如神奇的Byte-by-Byte-Copy), 基本上就差不多了. 如果说ACCEL那个阶段还有什么优化的话就是我把libm改成了libimf, 本质上和AVX2也差不多, 我不相信Intel咩有用AVX2优化他们的Basic Math-Lib(不过我们集群没有AVX-512支持, 所以没办法搞AVX512), 不过这就表示我偷懒了. 算是比较简单的优化, 如果算上直接在tmpfs文件系统中跑整个数据的话, 就这也拿了个小第一. 就这样我们稀里糊涂就进了总决赛, 算是保住了SCT的香火, 至少没有在我们这届手里凉掉:smile:.   

### 出发前 & Day0

出发前, 我们准备了8块A100, 非常遗憾的是我们没有设备去插这些A100, 于是我们的另一位指导老师陈博士和浪潮那边Social了一波, 借到了4台机器. 其中两台机器很早就拿回来插GPU, 但是另外两台机器直到出发前一天才上电, 因此实际上我们没有时间按照最早的构想测试4个节点的HPL和HPCG. 其实我在比赛前大概一周的时候犯了一个非常大的错误, 我使用八卡的配置去跑我们的节点, 一直没有在总共四卡的两个节点上跑起来. 最后我们有机会测试八卡配置的时候依然没有正常起来, 当时我以为是Infiniband的问题, 后来才发现实际上是后面两个节点的CPU配置和前面两个节点的配置不同, 因此配置NUMA时相同的配置后面两个节点会找不到Core. 最后在凌晨一点发现了这个情况, 还是勉强跑起来了.

出发前那天晚上我跳跳鱼二极管基本上没睡觉, 最后的结果就是那天早上非常困, 希望明年小朋友不要在出发前一天的晚上还在调试这种东西.

Day0 我们早上出发下午到酒店放了下东西就去拍照, 非常经典的看树上的花, 拍照的时候自己感觉像个憨批一样233.

晚上我们讨论了一下第二天的集群方案(虽然讨论的方案和实际的方案完全不一致), 陈博士希望我们使用更多的节点, 决赛报配置的时候甚至报了10台机器, 我当时就说不行的, 但是陈博士不信, 最后妥协下我们决定先上六个节点试一下.

### Day1 & Day2

正式比赛那天我们不知道以太网交换机可以直接插在排插上供电, 一开始插在PDU上, 直到某个工作人员跑过来说你们的以太网交换机可以插在排插上. 我们在赛前准备直接使用PDU做精细级的功耗控制, 甚至exporter都写好了, 然后我们拔掉PDU的网线之后他们就跑过来说你们不能这么做, 简直离谱, 我当时啊啪就起来了, 很快啊, 疯狂怼那个工作人员, 然后怼输了太丢人了, 总之rush最后写了一个新的exporter用来查看功耗数据, 数据直接从赛方那边采集, 慢了点但是又不是不能用. 

我的架构方案和家里那边差不多, 采用纯diskless的结构, 直接把rootfs挂在NFS上. 然后`/home` `/opt` `/mnt` 都是export到Slave计算节点上的. 这种方案实际上并不是那么的好使, 在docker, slurm等使用上会因为NFS无法支持完整的内核文件操作类型而报错, 如果以后还有的话这种方案一定要放到最后.

今年赛方采用的是ConnectX-6 HDR的Infiniband, 讲道理CX-6HDR的Infiniband非常好用, 但是赛放给了一个40口的Infiniband交换机, 交换机的功耗实在是太高了, 有亿点压不住. 我们最初的集群方案是6机8卡, 但是每台机器的静态功耗都将近400w, 最后依然采用4机8卡的集群架构. 我们犯的另外一个错误是拆掉了太多的内存, 虽然128G的内存可以大幅度降低功耗, 但是这也导致最后PRESTO最大那组数据不存在跑的可能性, QuEST后两组数据也不足以放在内存或显存中. 其间还有一个插曲, 我插内存的时候发现一个诡异的问题, 一般来说, 内存直接按照说明上的顺序对称安装就可以, 但是m6的固件只写了一半, 导致内存安装顺序是1-3-5-7, 否则内存设备可以被FM发现, 但是FM无法读取内存的容量.
