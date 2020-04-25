#### Chromium MessagePump
本文收益
- 旧版和新版Chromium MessagePump任务调度策略的主要逻辑
- 主要介绍OnNonDelayedLooperCallback和OnDelayedLooperCallback的逻辑


MessagePump/MessageLoop(简单)  
MessagePump的delegate是MessageLoop  
MessageLoop维护四个任务队列:  
`Incomming Queue`: PostTask任务首先进入InComming Queue  
`Work Queue`: 任务调度从Work Queue里面获取任务并执行. 当Work Queue队列为空时, 通过交换InComming Queue和Work Queue指针方式完成Incomming Queue任务装载到Work Queue过程.   
`Delayed Queue`:保存需要延时的任务. 每个从Work Queue获取的任务均需要判断是否需要延时, 若需要延时则加入延时队列. Delayed Queue是优先级队列, 先触发的延时任务排在队首  
`Defer Queue`: 这个队列通常是存放哪些因为MessageLoop嵌套而不能执行的任务，这些任务通常会在空闲时从defer队列中继续执行被defer的任务  

`MessagePump`负责任务调度, 它维护两个fd: NonDelayFd和TimeFd, 两个fd数据可读时可以唤醒epoll_wait. 当有非延迟任务可调度时, 通过Schedule Work往NonDelayFd写数据,  触发NonDelayedLooperCallback; ScheduleDelayWork往系统注册延时任务回调时间, 当到了延时任务触发时间时, TimeFd数据可读,, 触发DelayedLooperCallback.  

#### OnNonDelayedLooperCallback
 触发OnNonDelayedLooperCallback时, MessagePump任务调度策略如下:  
 ![image][https://github.com/leekeiling/PicturePool/blob/master/pics/messagepump1.png?raw=true]
 在OnNonDelayedLooperCallback, DoWork和DoDelayedWork以轮转法方式交替执行.   

**DoWork和DoDelayedWork做了啥?**  
DoWork 从 WorkQueue里面拿出任务, 判断任务是否为延时任务, 若是则加入到Delayed Queue里面．加入Delayed Queue之后需要判断加入的任务是否比其他延时任务更早触发, 若是则更新注册在系统中的定时timefd    
DoDelayedWork 从 Delayed Queue里面获取任务并判断在Delayed Queue里面的任务是否立即执行.   
若DoDelayedWork和DoWork在一个轮转周期均无实际执行任务, 则MessagePump跳出轮转并进入判断此刻Native是否处于Idle状态流程.   

**怎么判断是否处于Idle状态?** 
  
判断Idle状态是很重要过程, 若此刻Native MessageLoop处于Idle状态, 主端PollOnce返回开始处理Java任务.   
判断Idle状态依赖NonDelayFd的数值. 当退出Round Robin时会read NonDelayedFd数据. 若NonDelayedFd为1, 认为native处于Idle状态, 开始DoIdleWork; 若NonDelayedFd大于1, 认为native有其他任务调度, 重新ScheduleWork并触发OnNonDelayedLooperCallback, 触发OnNonDelayedLooperCallback之后, 从上述流程来看, 会查询Incoming Queue, Work Queue和DelayedWork Queue进行任务调度,    

ScheduleWork作用是往NonDelayedFd写数据1以期触发OnNonDelayedLooperCallback. 因此, 在Round Robin的时候其他线程调用ScheduleWork, 那么退出Round Robin并清空NonDelayedFd时候会发现NonDelayedFd大于1, 进而说明其他线程向Message Loop抛了任务, 此时native并不处于Idle状态.   
DoIdleWork做了啥?    

DoIdleWork清空Defer Queue里面的pending任务, 然后ScheduleDelayedWork向系统注册最近delayed work的触发时机.   

#### OnDelayedLooperCallback
触发OnDelayedLooperCallback时, MessagePump任务调度策略如下:  
<pic2>
首先是DoDelayedWork从Delayed Queue里面执行DelayedWork. 流程中MoreDelayedWork判断是否有其他延时任务, 若有则更新timeFd.  

最终都需要调用ScheduleWork, 为啥?  

ScheduleWork向NonDelayedFd写数据并触发OnNonDelayedLooperCallback, 只有OnNonDelayedLooperCallback里面可以判断native层是否Idle. 因此调用ScheduleWork是为了检测native层是否处于Idle状态.  

#### essagePump/ThreadControllerWithMessagePump(复杂)
MessagePump的delegate是ThreadControllerWithMessagePump  

#### OnNonDelayedLooperCallback
OnNonDelayedLooperCallback流程与上面一致, 不过此时将DoDelayedWork和DoWork方法合并成DoSomeWork, 将更多工作交给delegate处理, 此时delegate为ThreadControllerWithMessagePump.   

DoSomeWork第一步从TaskResource拿出Task来执行, TaskResource由SequenceManager管理.  

SequenceManager对Task细粒度控制, 同时提供Selector从任务队列中取出Task。  

Selector主要负责任务调度。  
<pic3>

结合代码过一遍上面流程～结合代码过一遍上面流程～  

首先是回调开始：  
<pic4>

回调结束：  
<pic5>

OnNonLooperCallback开始，首先清空唤醒此次callback的non_delayed_fd描述符，读取的值保存到pre_work_value.   

回调收尾阶段，若pre_work_value与kTryNativeTasksBeforeIdleBit不相等，说明其他对象调用ScheduleWork写入其他数据到non_delayed_fd， 意味着此次回调有任务调度； 若相等，说明此次回调仅仅是由kTryNativeTasksBeforeIdleBit数据起作用，此次回调没有任务调度，MessagePump处于Idle状态。  

举个栗子：  
- 调用 ScheduleWork， non_delayed_fd写入1
- non_delayed_fd有数据，OnNonLooperCallback回调开始，读non_delayed_fd，pre_work_value = 1
- OnNonLooperCallback回调收尾，pre_work_value != kTryNativeTasksBeforeIdleBit, 说明此次回调由ScheduleWork作用
- 往non_delayed_fd写入kTryNativeTasksBeforeIdleBit， return
- non_delayed_fd有数据，OnNonLooperCallback回调开始，读non_delayed_fd，pre_work_value = kTryNativeTasksBeforeIdleBit
- 假如某个对象有调用ScheduleWork， 那么pre_work_value = kTryNativeTasksBeforeIdleBit | 1， 此处跳过这个假设
- OnNonLooperCallback回调收尾，pre_work_value == kTryNativeTasksBeforeIdleBit, 说明此次回调仅仅由kTryNativeTasksBeforeIdleBit起作用，messagepump处于Idle状态

接下来是处理native任务：  
<pic6>

在while循环里，delegate_是ThreadControllerWithMessagePump对象，DoSomeWork处理$(batch size)数量的delayed task或者immediate task。  

**DoSomeWork**  
<pic7>  
DoSomeWork在for循环处理work_batch_size数量的任务, 它从task_source取出任务, 由task_annotator代理执行任务.  
任务调度策略主要在task_source->TakeTask()方法, task_source是SequenceManager对象.  

**TaskTask**  
<pic8>

主要逻辑如下:  
- ReloadEmptyWorkQueues将所有空WorkQueue与IncommingQueue交换指针,逻辑上将IncommingQueue的任务装载到WorkQueue
- MoveReadyDelayedTasksToWorkQueues
- 在while循环内通过selector.SelectWorkQueueToService选择一个WorkQueue, 以下重点讲selector.SelectWorkQueueToService 
- 依次判断队首任务是否取消, 是否处于嵌套状态, 条件不满足将重新通过selector选择另一个WorkQueue
- 返回队首任务

**selector**  
selector维护DelayedWorkQueueSet和ImmediateWorkQueueSet两种类型的多优先级任务队列集合, 多优先级任务队列集合架构如下(以DelayedWorkQueueSet为例):  
DelayedWorkQueueSet  
  - kControlPriority Heap : {q1, q2, q3, q4.. }
  - kHighestPriority Heap : {q1, q2, q3, q4.. }
  - kVeryHighPriority Heap: {q1, q2, q3, q4.. }
  - kHighPriority Heap: {q1, q2, q3, q4.. }
  - kNormalPriority Heap: {q1, q2, q3, q4.. }
  - kLowPriority Heap: {q1, q2, q3, q4.. }
  - kBestEffortPriority Heap: {q1, q2, q3, q4.. }

可以看到, DelayedWorkQueueSet和ImmediateWorkQueueSet下有7种优先级的堆, 其中, kControlPriority优先级最高, 优先执行并可以饿死其他队列; kBestEffortPriority优先级最低, 只有其他类型队列都执行完之后才开始执行.  每个堆下又挂载多个WorkQueue(q1, q2, q3...), 这些WorkQueue按任务的oldest程度排序.  
因此, selector任务调度首先从QueueSet中选择某种优先级的Heap, 然后在这个Heap中选择包含oldest任务的WorkQueue, 最后从WorkQueue取出队首任务.  

**SelectWorkQueueToService**  
<pic9>

SelectWorkQueueToService首先从active_priorities中获取下一个优先级类型. active_priorites保存目前活跃的优先级种类, active_priorities按每个优先级的key值大小排序, 下一次优先级种类的选择依据最小key值.  
第二步更新selection_count值, selection_count参与优先级key大小的更新.   
第三步更新priority key值大小并重新调整在active_priorities的排序. 每次从从active_priorities中选择priority, 需要重新更新该priority的key值大小以及在active_priorities的排序.   
key大小的更新实现在GetSortKeyForPriority中,   
<pic10>

动态更新priority key值大小的原因是防止其他低优先级队列被高优先级队列饿死(kControlPriority和kBestEffortPriority除外).  

上文selector提到, 当我们确定选择priority种类时, 就可以从DelayedWorkQueueSet或者ImmediateWorkQueueSet中选择对应的优先级种类的WorkQueue堆, 然后从该WorkQueue中选择包含oldest任务的WorkQueue.  

现在的问题是我们确定了priority, 现在是DelayedWorkQueueSet和ImmediateWorkQueueSet的选择问题. 第四步ChooseWithPriority有对应的策略实现, ChooseWithPriority函数实现如下:  
<pic11>

immediate_starvation_count标记选择DelayedWorkQueueSet的次数, 当immediate_starvation_count > kMaxDelayedStarvationTasks, 说明已经开始饿死ImmediateWorkQueueSet, 将优先选择ImmediateWorkQueueSet.    
假如还达不饿死ImmediateWorkQueueSet条件, 选择哪个WorkQueueSet由ChooseImmediateOrDelayedTaskWithPriority方法决策.  

<pic12>

在ChooseImmediateOrDelayedTaskWithPriority中, GetWithPriorityAndroidEnqueueOrder是从WorkQueueSet中选择WorkQueue以及WorkQueue插入到集合时的序号.    

ChooseImmediateOrDelayedTaskWithPriority会从ImmediateWorkQueueSet和DelayedWorkQueueSet分别选择对应的WorkQueue, 选择插入序号最小的WorkQueue.   

到这里, TakeTask到SelectWorkQueueToService的主要逻辑已经完了, 主要是SequenceManager的selector对象如何管理多优先级队列集合和任务调度策略。  
从TaskSource中选取到WorkQueue之后，就是从WorkQueue队首获取任务closure，最后是RunTask。  

#### OnDelayedLooperCallback
<pic13>
在分析OnNonDelayedLooperCallback基础上，OnDelayedLooperCallback逻辑并不复杂。  
  - 首先清空delayed_fd数据
  - delegate调用DoSomeWork；DoSomeWork的流程在前面已经分析过了
  - 如果DoSomeWork返回的下一次任务是Immediate的，那么SheduleWork往non_delayed_fd写数据，希望触发OnNonDelayedLooperCallback
  - 若下一次任务不是Immdediate，DoIdleWork处理defer队列的任务
  - ScheduleDelayedWork向系统注册下一次延迟任务的回调时机

#### 总结
两个MessagePump的主要区别是Deleagte不同,由Delegate不同而有不同的任务调度策略. 第一个MessagePump的任务调度策略是简单的在DelayedQueue和NonDelayedQueue轮转, 第二个MessagePump的是多优先级多队列的公平调度策略.



