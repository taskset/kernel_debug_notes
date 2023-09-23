
# 一、调度延迟分析


## 一）、Top N 延迟

目的是找出最大的TOP N 延迟及其根因。

```
select ts.ts, ts.dur, ts.utid, th.tid, th.name, ss.cpu, ss.priority
from thread_state as ts  left join thread as th using (utid) left join sched_slice as ss on (ts.ts + ts.dur = ss.ts)
where ts.state = 'R' and ts.dur >= 100000 
order by ts.dur desc

```

调度延迟从大到小排序如下：

![avatar](./img/max_delay.png)

分别分析这些case。

### 1、内核线程ksoftirq、rcu等延迟了40ms+

怀疑是被某一个实时进程所阻塞，继续分析：
```
select * from 
sched_slice left join thread using (utid)
where 
(ts + dur) >= 697901833440 and (ts + dur) <= (697901833440 + 47087488) and cpu = 3
```

被阻塞的时间段内，该cpu上的执行情况如下：
![avatar](./img/usage_cpu3.png)

可见，高优先级的实时线程cashm_dispather执行了45ms，堵塞了这个cpu上的所有其他线程。

回到perfetto，仔细展开对应的线程和cpu，也可以直观的看：
![avatar](./img/usage_cpu3_img.png)


### 2、recvMC线程延迟了40ms+

分析被阻塞的时间段内，该cpu上的执行情况如下：

```
select * from 
sched_slice left join thread using (utid)
where 
(ts + dur) >= 695896678176  and (ts + dur) <= (695896678176  + 43361920) and cpu = 8
order by dur desc

```

占用cpu时间从大到小排列，如图：
![avatar](./img/usage_cpu8.png)

主要是业务线程都集中在这个核上，超过1ms的线程需要关注，一些sh脚本也需要关注（时延敏感的系统所在的cpu上，不应该存在重量级的shell脚本）。
还有一点值得注意：上一步看到的实时线程也在这个核上。


### 3、imgman_thrpl_0线程延迟37ms+

分析被阻塞的时间段内，该cpu上的执行情况如下：

```
select * from 
sched_slice left join thread using (utid)
where 
(ts + dur) >= 695902758592  and (ts + dur) <= (695902758592  + 37910784) and cpu = 8
order by dur desc

```

占用cpu时间从大到小排列，如图：
![avatar](./img/usage_cpu8_thrpl.png)

超过1ms的线程需要关注，一些sh脚本也需要关注（时延敏感软件所运行的cpu上，不应该存在重量级的shell脚本）。上一步看到的实时线程在这个核上也在跑。


### 3、CamAbs_10_RD延迟30ms+
分析被阻塞的时间段内，该cpu上的执行情况如下：

```
select * from 
sched_slice left join thread using (utid)
where 
(ts + dur) >= 695904417728  and (ts + dur) <= (695904417728  + 35608352) and cpu = 3
order by dur desc

```
占用cpu时间从大到小排列，如图：
![avatar](./img/usage_cpu3_camabs.png)

调度延迟只有35ms，但是长期占据cpu的cashm却运行了40ms+，是不是分析得不对呢？

专门再去展开对应线程&cpu的图，确认是一致的：
![avatar](./img/usage_cpu3_cashm.png)


## 二）、FREQ延迟

目的是找出哪些进程或者哪些CPU容易出现延迟。

```
create view cpu_thread_delay as 
select ts.ts, ts.dur, ts.utid, th.tid, th.name, ss.cpu, ss.priority
from thread_state as ts  left join thread as th using (utid) left join sched_slice as ss on (ts.ts + ts.dur = ss.ts)
where ts.state = 'R' 
```

- 分析调度延迟和CPU的关系
  
![avatar](./img/cnt_10us.png)

---》 大于10us的调度延迟，CPU0 最多，原因是中断都集中在CPU0上。

![avatar](./img/cnt_100us.png)

---》 大于100us的调度延迟，CPU11 最多，和CPU11 的特殊业务有关。
 
![avatar](./img/cnt_10ms.png)

---》 大于10ms的调度延迟，CPU3 最多，和CPU3 上的实时线程有关系； cpu 11没有大于10ms的调度延迟，说明这个CPU上的业务还是符合预期。
 
![avatar](./img/cnt_40ms.png)

---》 大于40ms的调度延迟，只在CPU3 和 CPU8 存在，和它们上面的实时线程、以及业务部署情况有关系。
 
![avatar](./img/cnt_45ms.png)

---》 大于45ms的调度延迟，只在CPU3存在，和实时线程有关系。
 

小结：
CPU0 上有中断，会引起频繁的、较小的调度延迟；
CPU11 可能是有特殊用途，有大量中等程度的调度延迟，但没有大的调度延迟；
CPU8 业务部署比较密，有较大的调度延迟；
CPU3 长期有实时业务，引起了最大的调度延迟。

