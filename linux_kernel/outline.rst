nptl创建的大纲
==================

1. kernel调度单位task

2. 下面是copy_process的流程

3. dup_task_struct: 复制task结构

5. 初始化信号队列(链表)

6. sched_fork, copy_sighand, 处理io, fs, file

7. 创建pid和pid_amespace

   pid namespace的层级和映射, clone参数是CLONE_NEWPID

   pid包含所有的pid namespace和pid号, 存储在upids里面

   pid的创建是新建pid结果, 其中pid namespace的idr机制(基数树), 然后pid号分配是cyclic(循环)分配, 从最后一个使用的pid号开始找第一个

   可用的pid号, 找不到从头开始找.

   然后设置设置upids数组

   pid namespace中, 分配pid号和attach到pid namespace是分两步

8. 设置tgid(pid号), thread_group, group_leader

9. init_pid/attach_pid: pid结构和task结构关联起来

epoll大纲
============

1. waitqueue机制和惊群, wait_queue_entry, add_wait_queue_exclusive, ep_poll_callback

2. poll_wait函数
   
3. 就绪链表readylist
   
4. 无锁赋值ovflist
   
5. 红黑树

6. epitem, default_wake_function

6. ET/LT区别

cfs调度大纲
===============

1. O(1)的数据结构, bitmap和runqueue, runqueue是fifo去获取下一个task

   根据平均睡眠时间去判断是否是io密集型task(计算过程是进入睡觉时间就增加, 运行就减少)

   针对io密集型task会提升优先级

2. cfs具有抢占性, 用红黑树存储vruntime, 选vruntime最小的task来运行
   
   思路是优先级高的vruntime增长慢, 优先级低的增加快, 能保证某一个task总会被运行

   优先级决定了load_weight, load_weight和运行时间决定了vruntime


3. sched_fork中, 先把调用__sched_for, 把task->vruntime, task->sum_exec_runtime/prev_exec_runtime置0

   然后调用cfs的task_fork_fair

   如果定义了child_run_first, 交换父子task的vruntime
   
4. task_fork_fair
   
   调用update_curr,  更新cfs->curr->vruntime和cfs->min_vruntime
   
   调用place_entity, 新的task的vruntime初始化为cfs->min_vruntime, 如果定义了新fork出来的task需要延迟执行, 那么进行vrumtime的补偿


5. update_curr中, curr->vruntime增加的值是now - exec_start = delta, 然后cfs->curr->vruntime += delta * (NICE_0_LOAD / se->load_weight)

   优先级越高, 除的值就越小, +=的值就越小, vruntime变化就越慢

6. update_min_vruntime, 保证cfs->min_runtime是一个合理的, 单调的值. max(min(leftmost, curr), min_vruntime)


7. place_entity会对新创建的task和唤醒的task的vruntime进行补偿

   vruntime = cfs->min_vruntime

   新创建并且设置了start_debit话, vrumtime += sched_vslice

   补偿的依据是该task在cfs中被分配的时间片:
   
   base_slice = nr_running * sysctl_sched_min_granularity if nr_running > sysctl_sched_nr_latency else sysctl_sched_latency

   然后计算task在cfs_rq中的load的占比, 获得最终的slice, slice = base_slice * (task->load_weight / cfs->load)

   然后根据slice, task的load_weight(和nice_0对比), 去计算最终的增加的vruntime: vruntime += slice * (NICE_0_LOAD / task->load_weight)


8. wake_up_new_task/default_wake_function

   前者是clone中去唤醒, 后者是当有时间通知的时候, 调用的回调

9. wake-up_new_task: activate_task, 加入cfs红黑树, check_preempt_curr会调用check_preempt_wakeup, 判断唤醒的task是否应该去抢占掉当前task

   default_wake_function -> ttwu_queue -> ttwu_do_active -> (ttwu_active, ttwu_do_wakeup),  ttwu_active会调用enqueue_task加入红黑树, 然后调用ttwu_do_wakeup, 也会调用到check_preempt_curr


10. check_preempt_wakeup, 判断传入(唤醒)的task是否抢占当前task, 调用cfs的check_preempt_wakup

    其中唤醒的task会根据配置优先设置到next/last中, 这样选择下一个task的时候, 会对比leftmost, curr, next/last, 选一个合适的

    而唤醒的task是否应该抢占curr, 是判断传入的task的vruntime是否足够小, 计算函数是wakeup_preempt_entity

    task->vruntime < curr->vrumtime, and task->vruntime < curr->vruntime + gran, gran = sysctl_sched_min_granularity * (NICE_0_LOAD/task->load_weight)


11. resched_curr, 把curr设置上TIF_NEED_RESCHED, 可能需要跨cpu

12. 做一次context_switch的地方是schedule(__schedule)函数, 比如epoll中休眠的时候, 放弃cpu对调用schedule函数


13. __schedule函数是pick_next_task, 调用到cfs的pick_next_task_fair, 对比curr, leftmost, next/last, 对比的过程是变量left=min(leftmost, curr),
    
    然后如果next有值, 那么调用wakeup_preempt_entity去校验next和left, 如果last有值, 然后调用wakeup_preempt_entity去校验last和left

14. 然后周期性的schedule_tick也会校验当前curr是否已经用完时间片了, 计算条件当前总运行时间是否大于被分配的运行时间

    sum_exec = sum_exec_runtime - prev_exec_runtime

    ideal = sched_slice = base_slice * (curr->load / cfs->load)

    sum_exec > ideal and sum_exec > sysctl_sched_min_granularity

    然后调用resched_curr

15. 处理中断, 然后返回用户态的时候, exit_to_usermode会检查TIF_NEED_RESCHED, 然后调用__schedule


信号处理大纲
================

1. 线程的信号共享

2. 信号加入到pendding链表(shared_pending)

3. 强制信号, 比如sigkill, 那么选一个, 否则把信号都所有的线程中

4. wants_signal去先选择主线程, 遍历再任意一个

5. 唤醒的时候会强制唤醒TASK_INTERRUPTIBLE状态的线程, 如果没去唤醒, 那么kick_process强制唤醒

6. 唤醒的时候, 设置thread_info中的TIF_SIGPENDING标志位, 然后把信号加入到pending链表中

5. 返回用户态的时候, exit_to_usermode, 判断TIF_SIGPENDING, 那么切换程序堆栈





