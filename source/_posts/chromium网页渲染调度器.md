---
title: 网页渲染调度器
date: 2020-01-23 13:46:09
tags: chromium
---
网页渲染调度器
<!--more-->
### vsnc信号时间轴
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/vsync.png"/>
</center>


一个vsync信号周期包括render进程渲染和browser进程合成, Deadline之前是渲染之外的准备工作
### Scheduler
在一个vsync信号周期, Scheduler需要协同render thread, compositor thread 和 browser thread
- Compositor thread 负责网页UI渲染, 输入是Render thread的输出
- Render thread 解析网页, 渲染CC Layer Tree输出网页UI内容给Compositor thread
- Browser thread  网页UI合成, 输入是Compositor thread的输出


### LayerTreeFrameSinkState
```C++
  enum LayerTreeFrameSinkState {
    LAYER_TREE_FRAME_SINK_NONE,
    LAYER_TREE_FRAME_SINK_ACTIVE,
    LAYER_TREE_FRAME_SINK_CREATING,
    LAYER_TREE_FRAME_SINK_WAITING_FOR_FIRST_COMMIT,
    LAYER_TREE_FRAME_SINK_WAITING_FOR_FIRST_ACTIVATION,
  };
```

FrameSinkState描述网页绘图表面的状态
- LAYER_TREE_FRAME_SINK_NONE 是绘图表面的最初状态, 表示网页绘图表面尚未创建
- LAYER_TREE_FRAME_SINK_CREATING  CC Layer Tree首次创建, 开始创建绘图表面
- LAYER_TREE_FRAME_SINK_WAITING_FOR_FIRST_COMMIT 绘图表面创建完毕, 通知Scheduler尽快同步CC Layer Tree到Compositor线程Pending Tree
- LAYER_TREE_FRAME_SINK_WAITING_FOR_FIRST_ACTIVATION , CC Layer Tree同步完毕, 通知Scheduler尽快激活Pending Tree


#### BeginImplFrameState
下一个VSync信号到来之前，状态机处于BEGIN_IMPL_FRAME_STATE_IDLE状态, 持续时间到Vsyn周期内DeadLine时间点
```C++
  enum BeginImplFrameState {
    BEGIN_IMPL_FRAME_STATE_IDLE,
    BEGIN_IMPL_FRAME_STATE_INSIDE_BEGIN_FRAME,
    BEGIN_IMPL_FRAME_STATE_INSIDE_DEADLINE,
  };
```
BEGIN_IMPL_FRAME_STATE_INSIDE_BEGIN_FRAME开始是CC Layer Tree的绘制

#### CC Layer Tree
```C++  
  enum BeginMainFrameState {
    BEGIN_MAIN_FRAME_STATE_IDLE,
    BEGIN_MAIN_FRAME_STATE_SENT,
    BEGIN_MAIN_FRAME_STATE_STARTED,
    BEGIN_MAIN_FRAME_STATE_READY_TO_COMMIT,
  };
```
Render端CC Layer的绘制和提交状态
- BEGIN_MAIN_FRAME_STATE_IDLE 初始状态
- BEGIN_MAIN_FRAME_STATE_SENT Schedule已经请求Render 线程对CC Layer Tree绘制
- BEGIN_MAIN_FRAME_STATE_STARTED  Render线程开始对 CC Layer Tree绘制
- BEGIN_MAIN_FRAME_STATE_READY_TO_COMMIT 表示CC Layer Tree需要同步到CC Pending Layer Tree中去

### ForcedRedrawOnTmieoutState
```C++
enum ForcedRedrawOnTimeoutState {
    FORCED_REDRAW_STATE_IDLE,
    FORCED_REDRAW_STATE_WAITING_FOR_COMMIT,
    FORCED_REDRAW_STATE_WAITING_FOR_ACTIVATION,
    FORCED_REDRAW_STATE_WAITING_FOR_DRAW,
  };
```
描述的网页渲染状态, 
FORCED_REDRAW_STATE_IDLE 
- 网页渲染状态为IDLE
FORCED_REDRAW_STATE_WAITING_FOR_COMMIT 
- 等待Render线程同步pending tree到Compositor线程
FORCED_REDRAW_STATE_WAITING_FOR_ACTIVATION
- 存在CC Pending Layer Tree, 等待被激活
FORCED_REDRAW_STATE_WAITING_FOR_DRAW
- 等待CC Ative Layer Tree被渲染

### Actions
```C++ 
 enum Action {
    ACTION_NONE,
    ACTION_SEND_BEGIN_MAIN_FRAME,
    ACTION_COMMIT,
    ACTION_ACTIVATE_SYNC_TREE,
    ACTION_PERFORM_IMPL_SIDE_INVALIDATION,
    ACTION_DRAW_IF_POSSIBLE,
    ACTION_DRAW_FORCED,
    ACTION_DRAW_ABORT,
    ACTION_BEGIN_LAYER_TREE_FRAME_SINK_CREATION,
    ACTION_PREPARE_TILES,
    ACTION_INVALIDATE_LAYER_TREE_FRAME_SINK,
    ACTION_NOTIFY_BEGIN_MAIN_FRAME_NOT_SENT,
  };
```
- ShouldActivateSyncTree
- ShouldCommit
- ShouldDraw
  - ACTION_DRAW_ABORT
  - ACTION_DRAW_FORCED
  - ACTION_DRAW_IF_POSSIBLE
- ShouldPerformImplSideInvalidation
- ShouldPrepareTiles
- ShouldSendBeginMainFrame
- ShouldInvalidateLayerTreeFrameSink
- ShouldBeginLayerTreeFrameSinkCreation
- ShouldNotifyBeginMainFrameNotSent
### 渲染流程
#### Vsnyc 
Vsync是Android系统底层发的中断通知, android系统每隔16.6ms发出VSYNC信号，来通知界面进行输入、动画、绘制等动作，每一次同步的周期为16.6ms，代表一帧的刷新频率

在Vsync信号里面, Browser thread向compositor thread发送ViewMsg_BeginFrame IPC消息, 通知cc thread BeginImplFrame, 对应scheduler状态机的ACTION_SEND_BEGIN_MAIN_FRAME动作
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/vsynctrace1.png"/>
</center>
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/vsynctrace2.png"/>
</center>

#### BeginMainFrame
ThreadProxy::BeginMainFrame 
Render thread绘制cc layer tree
- Parse
- Layout
- Paint
- Commit
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/beginmainframe.png"/>
</center>

#### Sync CC Layer Tree
ProxyImpl::NotifyReadyToCommitOnImpl
Render thread绘制好cc layer tree之后, 向cc thread抛NotifyReadyToCommitOnImpl通知, cc thread接下来跟render thread同步cc layer tree, 最终在cc thread有一个pending layer tree

对应的scheduler动作为: ACTION_COMMIT
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/synccc.png"/>
</center>

#### Raster
光栅化在CompositorTileWorker线程工作, 这样做的好处是cc thread可以快速响应处理ui线程的onDraw命令,并且及时更新scheduler状态机, 跟render thread同步状态等

对应的scheduler动作为: ACTION_PREPARE_TILES
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/raster.png"/>
</center>

#### Active
分块Tiles光栅化结束后, 需要对cc thread端的pending layer tree激活为active tree. Active layer tree是用户真正看到的屏幕内容

对应的scheduler动作为:
```C++
 ACTION_DRAW_IF_POSSIBLE,
 ACTION_DRAW_FORCED,
 ACTION_DRAW_ABORT
```
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/active.png"/>
</center>
如果render thread在同步cc layer tree的时候有向cc thread注册swap通知(比如fcp swap), 那么这时候会调用swap->DidActive, 通知pending tree已经被激活

#### OnDraw
在Browser线程, Awcontent调用OnDraw, 发送SyncCompositorMsg_DemandDrawHw ipc消息, 通知cc thread绘制网页ui

<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/ondraw.png"/>
</center>
```C++
AwContents.onDraw
```
Cc thread收到SyncCompositorMsg_DemandDrawHw通知后, 开始对active layer 绘制渲染, 最终把RenderPass传给browser thread
<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/ondraw2.png"/>
</center>
Cc thread最终将gpu 命令写入gl in-process command buffer. 如果render thread在同步cc layer tree的时候有向cc thread注册swap通知(比如fcp swap), 那么这时候会调用swap->DidSwap

#### DrawFunctor
DrawFunctor是启动硬件加速渲染情况下, 由android系统主动回调过来; 在DrawFunctor里面, 最终通过SwapBuffer动作把gl in-processs command buffer里面的gpu 命令转发给app render线程. SwapBuffer动作完成之后, 才是真正的上屏动作

<center>
    <img src="https://raw.githubusercontent.com/leekeiling/PicturePool/master/pics/drawfunctor.png"/>
</center>
