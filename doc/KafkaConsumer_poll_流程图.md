# KafkaConsumer poll() 方法全链路流程图

## 1. poll() 方法主流程

```mermaid
flowchart TD
    Start([用户调用 poll]) --> Acquire[acquireAndEnsureOpen<br/>获取锁并检查状态]
    Acquire --> CheckSub[检查是否有订阅或分配]
    CheckSub --> Loop{do-while循环<br/>直到超时}
    
    Loop --> Wakeup[client.maybeTriggerWakeup<br/>检查是否需要唤醒]
    Wakeup --> UpdateMeta{includeMetadataInTimeout?}
    
    UpdateMeta -->|是| UpdateMeta1[updateAssignmentMetadataIfNeeded<br/>更新元数据但不阻塞]
    UpdateMeta -->|否| UpdateMeta2[updateAssignmentMetadataIfNeeded<br/>阻塞等待元数据]
    
    UpdateMeta1 --> PollFetch[pollForFetches<br/>拉取消息]
    UpdateMeta2 --> PollFetch
    
    PollFetch --> HasRecords{records.isEmpty?}
    HasRecords -->|否| SendNextFetch[发送下一轮fetch请求<br/>实现pipelining]
    SendNextFetch --> ReturnRecords[返回ConsumerRecords]
    
    HasRecords -->|是| CheckTimeout{timeout未过期?}
    CheckTimeout -->|是| Loop
    CheckTimeout -->|否| ReturnEmpty[返回空ConsumerRecords]
    
    ReturnRecords --> Release[release<br/>释放锁]
    ReturnEmpty --> Release
    Release --> End([结束])
    
    style PollFetch fill:#e1f5ff
    style UpdateMeta1 fill:#fff4e1
    style UpdateMeta2 fill:#fff4e1
```

## 2. updateAssignmentMetadataIfNeeded 详细流程（包含心跳）

```mermaid
flowchart TD
    Start([updateAssignmentMetadataIfNeeded]) --> CheckCoord{coordinator != null?}
    
    CheckCoord -->|是| CoordPoll[coordinator.poll<br/>协调器轮询]
    CheckCoord -->|否| UpdatePos[updateFetchPositions<br/>更新拉取位置]
    
    CoordPoll --> MaybeUpdateSub[maybeUpdateSubscriptionMetadata<br/>更新订阅元数据]
    MaybeUpdateSub --> InvokeCallbacks[invokeCompletedOffsetCommitCallbacks<br/>调用offset提交回调]
    InvokeCallbacks --> CheckAutoAssign{hasAutoAssignedPartitions?}
    
    CheckAutoAssign -->|是| PollHeartbeat[pollHeartbeat<br/>轮询心跳]
    CheckAutoAssign -->|否| UpdatePos
    
    PollHeartbeat --> CheckThread{heartbeatThread != null?}
    CheckThread -->|是| CheckFailed{heartbeatThread.hasFailed?}
    CheckFailed -->|是| ThrowException[抛出异常]
    CheckFailed -->|否| CheckShouldHB{heartbeat.shouldHeartbeat?}
    
    CheckShouldHB -->|是| NotifyThread[notify心跳线程<br/>唤醒心跳线程]
    CheckShouldHB -->|否| UpdateHeartbeat[heartbeat.poll<br/>更新心跳时间]
    
    NotifyThread --> UpdateHeartbeat
    UpdateHeartbeat --> CheckCoordReady{coordinatorUnknown?}
    
    CheckCoordReady -->|是| EnsureCoord[ensureCoordinatorReady<br/>确保协调器就绪]
    CheckCoordReady -->|否| CheckRejoin{rejoinNeededOrPending?}
    
    EnsureCoord -->|失败| ReturnFalse[返回false]
    EnsureCoord -->|成功| CheckRejoin
    
    CheckRejoin -->|是| EnsureActive[ensureActiveGroup<br/>确保组活跃]
    CheckRejoin -->|否| UpdatePos
    
    EnsureActive -->|失败| ReturnFalse
    EnsureActive -->|成功| UpdatePos
    
    UpdatePos --> ReturnTrue[返回true]
    
    style PollHeartbeat fill:#ffcccc
    style NotifyThread fill:#ffcccc
    style UpdateHeartbeat fill:#ffcccc
```

## 3. 心跳线程工作机制

```mermaid
flowchart TD
    Start([心跳线程启动]) --> Loop{while true循环}
    
    Loop --> CheckClosed{closed?}
    CheckClosed -->|是| Exit([退出])
    CheckClosed -->|否| CheckEnabled{enabled?}
    
    CheckEnabled -->|否| Wait1[wait等待唤醒]
    Wait1 --> Loop
    
    CheckEnabled -->|是| CheckJoined{hasNotJoinedGroup<br/>或hasFailed?}
    CheckJoined -->|是| Disable[disable心跳线程]
    Disable --> Loop
    
    CheckJoined -->|否| ClientPoll[client.pollNoWakeup<br/>处理网络IO]
    ClientPoll --> GetTime[获取当前时间now]
    GetTime --> CheckCoordUnknown{coordinatorUnknown?}
    
    CheckCoordUnknown -->|是| FindCoord[查找协调器]
    FindCoord --> Wait2[wait重试退避时间]
    Wait2 --> Loop
    
    CheckCoordUnknown -->|否| CheckSessionTimeout{sessionTimeoutExpired?}
    CheckSessionTimeout -->|是| MarkUnknown[标记协调器未知]
    MarkUnknown --> Loop
    
    CheckSessionTimeout -->|否| CheckPollTimeout{pollTimeoutExpired?}
    CheckPollTimeout -->|是| LeaveGroup[maybeLeaveGroup<br/>离开消费者组]
    LeaveGroup --> Loop
    
    CheckPollTimeout -->|否| CheckShouldHB{heartbeat.shouldHeartbeat?}
    CheckShouldHB -->|否| Wait3[wait重试退避时间]
    Wait3 --> Loop
    
    CheckShouldHB -->|是| SentHB[heartbeat.sentHeartbeat<br/>标记心跳已发送]
    SentHB --> SendHB[sendHeartbeatRequest<br/>发送心跳请求]
    
    SendHB --> AddListener[添加响应监听器]
    AddListener --> OnSuccess{心跳成功?}
    
    OnSuccess -->|是| ReceiveHB[heartbeat.receiveHeartbeat<br/>接收心跳响应]
    OnSuccess -->|否| FailHB[heartbeat.failHeartbeat<br/>心跳失败]
    
    ReceiveHB --> Loop
    FailHB --> Notify[notify唤醒]
    Notify --> Loop
    
    style SendHB fill:#ffcccc
    style SentHB fill:#ffcccc
    style ReceiveHB fill:#ffcccc
```

## 4. pollForFetches 详细流程（包含CompletedFetch机制）

```mermaid
flowchart TD
    Start([pollForFetches]) --> CalcTimeout[计算pollTimeout<br/>考虑coordinator下次poll时间]
    
    CalcTimeout --> TryCache[fetcher.fetchedRecords<br/>尝试从缓存获取数据]
    TryCache --> HasCacheData{records.isEmpty?}
    
    HasCacheData -->|否| ReturnCache[返回缓存的records]
    HasCacheData -->|是| SendFetches[fetcher.sendFetches<br/>发送新的fetch请求]
    
    SendFetches --> PrepareFetch[prepareFetchRequests<br/>准备fetch请求]
    PrepareFetch --> ForEachNode{遍历每个Node}
    
    ForEachNode --> BuildRequest[构建FetchRequest]
    BuildRequest --> AsyncSend[client.send异步发送请求]
    AsyncSend --> AddListener[添加响应监听器]
    
    AddListener --> OnFetchSuccess{响应成功?}
    OnFetchSuccess -->|是| ParseResponse[解析FetchResponse]
    OnFetchSuccess -->|否| HandleError[处理错误]
    
    ParseResponse --> ForEachPartition{遍历每个分区}
    ForEachPartition --> CreateCompletedFetch[创建CompletedFetch对象]
    CreateCompletedFetch --> AddToQueue[completedFetches.add<br/>添加到队列]
    
    AddToQueue --> ForEachNode
    HandleError --> ForEachNode
    
    ForEachNode -->|完成| ClientPoll[client.poll等待响应<br/>直到有可用数据或超时]
    
    ClientPoll --> FetchRecords[fetcher.fetchedRecords<br/>从缓存队列获取数据]
    FetchRecords --> ReturnRecords[返回records]
    
    style TryCache fill:#ccffcc
    style AddToQueue fill:#ccffcc
    style FetchRecords fill:#ccffcc
```

## 5. CompletedFetch 异步拉取和缓存机制详解

```mermaid
flowchart LR
    subgraph "异步拉取阶段"
        A1[sendFetches发送请求] --> A2[网络异步IO]
        A2 --> A3[响应到达回调]
        A3 --> A4[创建CompletedFetch]
        A4 --> A5[completedFetches队列]
    end
    
    subgraph "缓存队列"
        B1[ConcurrentLinkedQueue<br/>completedFetches]
        B2[线程安全的队列<br/>支持并发访问]
    end
    
    subgraph "消费阶段"
        C1[fetchedRecords调用] --> C2[从队列peek CompletedFetch]
        C2 --> C3{notInitialized?}
        C3 -->|是| C4[initializeCompletedFetch<br/>初始化并解析]
        C3 -->|否| C5[直接使用]
        C4 --> C6[fetchRecords解析成ConsumerRecord]
        C5 --> C6
        C6 --> C7[更新offset位置]
        C7 --> C8[返回给用户]
    end
    
    A5 --> B1
    B1 --> C2
    
    style A5 fill:#ccffcc
    style B1 fill:#ffffcc
    style C2 fill:#ffccff
```

## 6. fetchedRecords 详细处理流程

```mermaid
flowchart TD
    Start([fetchedRecords]) --> InitVars[初始化变量<br/>fetched, pausedCompletedFetches<br/>recordsRemaining = maxPollRecords]
    
    InitVars --> Loop{while recordsRemaining > 0}
    
    Loop --> CheckNext{nextInLineFetch == null<br/>或isConsumed?}
    
    CheckNext -->|是| PeekQueue[completedFetches.peek<br/>从队列peek一个CompletedFetch]
    PeekQueue --> CheckNull{records == null?}
    
    CheckNull -->|是| BreakLoop[break循环]
    CheckNull -->|否| CheckInit{records.notInitialized?}
    
    CheckInit -->|是| TryInit[try initializeCompletedFetch]
    TryInit --> InitSuccess{初始化成功?}
    InitSuccess -->|是| SetNext[nextInLineFetch = 初始化结果]
    InitSuccess -->|否| CheckEmpty{fetched.isEmpty<br/>且无实际内容?}
    
    CheckEmpty -->|是| PollQueue[completedFetches.poll<br/>移除无效fetch]
    CheckEmpty -->|否| ThrowException[抛出异常]
    
    CheckInit -->|否| SetNext2[nextInLineFetch = records]
    
    SetNext --> PollQueue
    SetNext2 --> PollQueue
    
    PollQueue --> CheckPaused{subscriptions.isPaused?}
    
    CheckPaused -->|是| AddPaused[pausedCompletedFetches.add<br/>保存暂停的分区数据]
    AddPaused --> ClearNext[nextInLineFetch = null]
    ClearNext --> Loop
    
    CheckPaused -->|否| FetchRec[fetchRecords解析记录]
    FetchRec --> HasRec{records.isEmpty?}
    
    HasRec -->|否| AddToFetched[添加到fetched Map]
    AddToFetched --> UpdateRemaining[recordsRemaining -= size]
    UpdateRemaining --> Loop
    
    HasRec -->|是| Loop
    
    BreakLoop --> Finally[finally块]
    ThrowException --> Finally
    
    Finally --> AddBackPaused[completedFetches.addAll<br/>pausedCompletedFetches<br/>将暂停的分区数据加回队列]
    AddBackPaused --> ReturnFetched[返回fetched Map]
    
    style PeekQueue fill:#ccffcc
    style PollQueue fill:#ccffcc
    style AddBackPaused fill:#ccffcc
```

## 7. CompletedFetch 内部结构和工作原理

```mermaid
flowchart TD
    subgraph "CompletedFetch 类结构"
        A1[partition: TopicPartition]
        A2[partitionData: FetchResponse.PartitionData]
        A3[batches: Iterator RecordBatch]
        A4[nextFetchOffset: long]
        A5[records: CloseableIterator Record]
        A6[isConsumed: boolean]
        A7[initialized: boolean]
    end
    
    subgraph "初始化过程 initializeCompletedFetch"
        B1[检查partition是否有有效位置]
        B1 --> B2{error == Errors.NONE?}
        B2 -->|是| B3[检查offset是否匹配]
        B3 --> B4{offset匹配?}
        B4 -->|是| B5[获取RecordBatch迭代器]
        B4 -->|否| B6[返回null丢弃]
        B5 --> B7[设置initialized = true]
    end
    
    subgraph "解析过程 fetchRecords"
        C1[nextFetchedRecord获取下一条记录]
        C1 --> C2{records == null<br/>或!hasNext?}
        C2 -->|是| C3{batches.hasNext?}
        C3 -->|是| C4[获取下一个RecordBatch]
        C3 -->|否| C5[drain清空并返回null]
        C4 --> C6[创建streamingIterator]
        C6 --> C1
        C2 -->|否| C7[records.next获取Record]
        C7 --> C8[验证并返回Record]
    end
    
    subgraph "drain过程"
        D1[关闭records流]
        D2[记录metrics]
        D3[移动partition到末尾]
        D4[设置isConsumed = true]
    end
    
    A3 --> B5
    B7 --> C1
    C5 --> D1
    
    style A3 fill:#ccffcc
    style B5 fill:#ffffcc
    style C1 fill:#ffccff
```

## 核心机制总结

### 1. 心跳机制
- **触发时机**: 在 `coordinator.poll()` 中通过 `pollHeartbeat()` 唤醒心跳线程
- **心跳线程**: 独立的后台线程，持续运行
- **发送逻辑**: 
  - 心跳线程检查 `heartbeat.shouldHeartbeat(now)` 
  - 如果到了心跳间隔时间，调用 `sendHeartbeatRequest()` 发送
  - 通过 `RequestFuture` 异步发送，响应通过监听器处理
- **关键点**: 
  - 心跳线程和主poll线程是分离的
  - 心跳失败会触发重试或离开组
  - 如果 `max.poll.interval.ms` 超时，心跳线程会主动离开组

### 2. CompletedFetch 异步拉取和缓存机制
- **异步拉取**: 
  - `sendFetches()` 异步发送 FetchRequest，不阻塞
  - 响应通过回调处理，创建 `CompletedFetch` 对象
  - `CompletedFetch` 被添加到 `completedFetches` 队列（`ConcurrentLinkedQueue`）
  
- **缓存机制**:
  - `completedFetches` 是线程安全的队列，支持并发访问
  - 响应可能在后台线程（如心跳线程）处理，添加到队列
  - 主线程调用 `fetchedRecords()` 时从队列取出数据
  
- **延迟解析**:
  - `CompletedFetch` 保存原始的 `FetchResponse.PartitionData` 和 `RecordBatch` 迭代器
  - 只有在 `fetchedRecords()` 时才真正解析成 `ConsumerRecord`
  - 通过 `nextInLineFetch` 缓存当前正在处理的 `CompletedFetch`
  
- **优势**:
  - **Pipelining**: 在返回当前批次数据前，可以发送下一轮fetch请求
  - **非阻塞**: 主线程不需要等待网络IO完成
  - **批量处理**: 一次fetch响应可能包含多个分区的数据，分批返回给用户

