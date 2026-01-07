# Q1: FreeRTOS 任務分配、初始化與加入就緒列表

## 1. 代碼追蹤流程（函數調用序列）

基於 FreeRTOS 源碼分析，以下是任務創建的完整函數調用序列：

```
使用者應用程式
    │
    ├─→ xTaskCreate()                          [tasks.c, 第 1718 行]
         │
         ├─→ prvCreateTask()                   [tasks.c, 第 1620 行]
         │    │
         │    ├─→ pvPortMallocStack()          [分配堆疊記憶體]
         │    │
         │    ├─→ pvPortMalloc()               [分配 TCB 記憶體]
         │    │
         │    └─→ prvInitialiseNewTask()       [tasks.c, 第 1793 行]
         │         │
         │         ├─→ memset()                [用已知值填充堆疊]
         │         │
         │         ├─→ vListInitialiseItem()   [初始化 StateListItem]
         │         │
         │         ├─→ vListInitialiseItem()   [初始化 EventListItem]
         │         │
         │         ├─→ listSET_LIST_ITEM_OWNER() [設定列表項擁有者]
         │         │
         │         └─→ pxPortInitialiseStack()  [port.c, 第 202 行]
         │              [ARM_CM4F 特定的堆疊設置]
         │
         └─→ prvAddNewTaskToReadyList()        [tasks.c, 第 2019 行]
              │
              ├─→ taskENTER_CRITICAL()         [禁用中斷]
              │
              ├─→ prvInitialiseTaskLists()     [僅第一個任務]
              │
              ├─→ prvAddTaskToReadyList()      [巨集，第 268 行]
              │    │
              │    ├─→ taskRECORD_READY_PRIORITY()
              │    │
              │    └─→ listINSERT_END()        [插入就緒列表]
              │
              ├─→ taskEXIT_CRITICAL()          [啟用中斷]
              │
              └─→ taskYIELD_ANY_CORE_IF_USING_PREEMPTION()
                   [必要時觸發情境切換]
```

---

## 2. 關鍵代碼片段

### 2.1 任務創建入口點
**檔案：** `tasks.c`  
**函數：** `xTaskCreate()`  
**行數：** 1718-1752

```c
BaseType_t xTaskCreate( TaskFunction_t pxTaskCode,
                        const char * const pcName,
                        const configSTACK_DEPTH_TYPE uxStackDepth,
                        void * const pvParameters,
                        UBaseType_t uxPriority,
                        TaskHandle_t * const pxCreatedTask )
{
    TCB_t * pxNewTCB;
    BaseType_t xReturn;

    // 步驟 1：分配並初始化 TCB
    pxNewTCB = prvCreateTask( pxTaskCode, pcName, uxStackDepth, 
                             pvParameters, uxPriority, pxCreatedTask );

    if( pxNewTCB != NULL )
    {
        // 步驟 2：將任務加入就緒列表
        prvAddNewTaskToReadyList( pxNewTCB );
        xReturn = pdPASS;
    }
    else
    {
        xReturn = errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY;
    }

    return xReturn;
}
```

### 2.2 TCB 與堆疊的記憶體分配
**檔案：** `tasks.c`  
**函數：** `prvCreateTask()`  
**行數：** 1620-1715

```c
static TCB_t * prvCreateTask( TaskFunction_t pxTaskCode,
                              const char * const pcName,
                              const configSTACK_DEPTH_TYPE uxStackDepth,
                              void * const pvParameters,
                              UBaseType_t uxPriority,
                              TaskHandle_t * const pxCreatedTask )
{
    TCB_t * pxNewTCB;
    
    // 針對 ARM Cortex-M4（堆疊向下增長，portSTACK_GROWTH < 0）
    StackType_t * pxStack;
    
    // 分配堆疊記憶體
    pxStack = pvPortMallocStack( ((size_t) uxStackDepth) * sizeof(StackType_t) );
    
    if( pxStack != NULL )
    {
        // 分配 TCB 記憶體
        pxNewTCB = ( TCB_t * ) pvPortMalloc( sizeof( TCB_t ) );
        
        if( pxNewTCB != NULL )
        {
            memset( (void *) pxNewTCB, 0x00, sizeof( TCB_t ) );
            pxNewTCB->pxStack = pxStack;  // 在 TCB 中儲存堆疊指標
        }
    }
    
    if( pxNewTCB != NULL )
    {
        // 初始化新任務
        prvInitialiseNewTask( pxTaskCode, pcName, uxStackDepth, pvParameters, 
                             uxPriority, pxCreatedTask, pxNewTCB, NULL );
    }
    
    return pxNewTCB;
}
```

### 2.3 任務控制塊初始化
**檔案：** `tasks.c`  
**函數：** `prvInitialiseNewTask()`  
**行數：** 1793-2014

```c
static void prvInitialiseNewTask( TaskFunction_t pxTaskCode,
                                  const char * const pcName,
                                  const configSTACK_DEPTH_TYPE uxStackDepth,
                                  void * const pvParameters,
                                  UBaseType_t uxPriority,
                                  TaskHandle_t * const pxCreatedTask,
                                  TCB_t * pxNewTCB,
                                  const MemoryRegion_t * const xRegions )
{
    StackType_t * pxTopOfStack;
    
    // 用已知值填充堆疊以進行除錯（0xa5）
    memset( pxNewTCB->pxStack, (int) tskSTACK_FILL_BYTE, 
            (size_t) uxStackDepth * sizeof(StackType_t) );
    
    // 計算堆疊頂端（針對向下增長的堆疊）
    pxTopOfStack = &( pxNewTCB->pxStack[ uxStackDepth - 1 ] );
    pxTopOfStack = (StackType_t *) ( ((portPOINTER_SIZE_TYPE) pxTopOfStack) & 
                   (~((portPOINTER_SIZE_TYPE) portBYTE_ALIGNMENT_MASK)) );
    
    // 儲存任務名稱
    for( x = 0; x < configMAX_TASK_NAME_LEN; x++ )
    {
        pxNewTCB->pcTaskName[ x ] = pcName[ x ];
        if( pcName[ x ] == 0x00 ) break;
    }
    
    // 設定優先權
    pxNewTCB->uxPriority = uxPriority;
    pxNewTCB->uxBasePriority = uxPriority;  // 用於優先權繼承
    
    // 初始化列表項
    vListInitialiseItem( &( pxNewTCB->xStateListItem ) );
    vListInitialiseItem( &( pxNewTCB->xEventListItem ) );
    
    // 將 TCB 設為列表項的擁有者
    listSET_LIST_ITEM_OWNER( &( pxNewTCB->xStateListItem ), pxNewTCB );
    listSET_LIST_ITEM_VALUE( &( pxNewTCB->xEventListItem ), 
                            (TickType_t) configMAX_PRIORITIES - (TickType_t) uxPriority );
    listSET_LIST_ITEM_OWNER( &( pxNewTCB->xEventListItem ), pxNewTCB );
    
    // 初始化堆疊以模擬中斷返回
    pxNewTCB->pxTopOfStack = pxPortInitialiseStack( pxTopOfStack, pxTaskCode, pvParameters );
    
    // 設定任務狀態
    pxNewTCB->xTaskRunState = taskTASK_NOT_RUNNING;
    
    // 返回任務控制代碼
    if( pxCreatedTask != NULL )
    {
        *pxCreatedTask = (TaskHandle_t) pxNewTCB;
    }
}
```

### 2.4 堆疊初始化（ARM Cortex-M4F 特定）
**檔案：** `portable/GCC/ARM_CM4F/port.c`  
**函數：** `pxPortInitialiseStack()`  
**行數：** 202-231

```c
StackType_t * pxPortInitialiseStack( StackType_t * pxTopOfStack,
                                     TaskFunction_t pxCode,
                                     void * pvParameters )
{
    // 模擬由情境切換中斷創建的堆疊框架
    pxTopOfStack--;
    
    *pxTopOfStack = portINITIAL_XPSR;        // xPSR 暫存器 (0x01000000)
    pxTopOfStack--;
    *pxTopOfStack = ((StackType_t) pxCode) & portSTART_ADDRESS_MASK;  // PC（任務進入點）
    pxTopOfStack--;
    *pxTopOfStack = (StackType_t) portTASK_RETURN_ADDRESS;  // LR（返回位址）
    
    pxTopOfStack -= 5;                       // R12, R3, R2, R1（未初始化）
    *pxTopOfStack = (StackType_t) pvParameters;  // R0（任務參數）
    
    pxTopOfStack--;
    *pxTopOfStack = portINITIAL_EXC_RETURN;  // 異常返回值 (0xFFFFFFFD)
    
    pxTopOfStack -= 8;                       // R11-R4（未初始化）
    
    return pxTopOfStack;
}
```

### 2.5 將任務加入就緒列表
**檔案：** `tasks.c`  
**函數：** `prvAddNewTaskToReadyList()`  
**行數：** 2019-2093

```c
static void prvAddNewTaskToReadyList( TCB_t * pxNewTCB )
{
    // 進入臨界區以保護任務列表
    taskENTER_CRITICAL();
    {
        uxCurrentNumberOfTasks++;
        
        // 如果這是第一個任務
        if( pxCurrentTCB == NULL )
        {
            pxCurrentTCB = pxNewTCB;
            
            if( uxCurrentNumberOfTasks == 1 )
            {
                // 初始化所有任務列表（就緒、延遲、暫停）
                prvInitialiseTaskLists();
            }
        }
        else
        {
            // 如果排程器未運行，在較高優先權時更新當前任務
            if( xSchedulerRunning == pdFALSE )
            {
                if( pxCurrentTCB->uxPriority <= pxNewTCB->uxPriority )
                {
                    pxCurrentTCB = pxNewTCB;
                }
            }
        }
        
        uxTaskNumber++;
        pxNewTCB->uxTCBNumber = uxTaskNumber;
        
        // 將任務加入適當的就緒列表
        prvAddTaskToReadyList( pxNewTCB );
        
        portSETUP_TCB( pxNewTCB );
    }
    taskEXIT_CRITICAL();
    
    // 如果排程器正在運行且任務優先權較高，則讓出 CPU
    if( xSchedulerRunning != pdFALSE )
    {
        taskYIELD_ANY_CORE_IF_USING_PREEMPTION( pxNewTCB );
    }
}
```

### 2.6 就緒列表插入巨集
**檔案：** `tasks.c`  
**巨集：** `prvAddTaskToReadyList()`  
**行數：** 268-274

```c
#define prvAddTaskToReadyList( pxTCB )                                                             \
    do {                                                                                           \
        traceMOVED_TASK_TO_READY_STATE( pxTCB );                                                   \
        taskRECORD_READY_PRIORITY( (pxTCB)->uxPriority );                                          \
        listINSERT_END( &(pxReadyTasksLists[ (pxTCB)->uxPriority ]), &((pxTCB)->xStateListItem) ); \
        tracePOST_MOVED_TASK_TO_READY_STATE( pxTCB );                                              \
    } while( 0 )
```

---

## 3. 解釋性圖表

### 3.1 任務控制塊（TCB）結構
```
TCB_t 結構（tasks.c, 第 358-436 行）：
┌────────────────────────────────────────────────────┐
│ pxTopOfStack          → 當前堆疊指標               │ ← 必須是第一個成員
├────────────────────────────────────────────────────┤
│ xStateListItem        → 連結任務至就緒/            │
│                         阻塞/暫停列表              │
├────────────────────────────────────────────────────┤
│ xEventListItem        → 連結任務至事件列表         │
├────────────────────────────────────────────────────┤
│ uxPriority           → 任務優先權（0=最低）        │
├────────────────────────────────────────────────────┤
│ pxStack              → 指向堆疊基底的指標          │
├────────────────────────────────────────────────────┤
│ xTaskRunState        → 運行/未運行狀態             │
├────────────────────────────────────────────────────┤
│ pcTaskName[16]       → 任務名稱字串                │
├────────────────────────────────────────────────────┤
│ pxEndOfStack         → 堆疊溢出檢測                │
├────────────────────────────────────────────────────┤
│ uxBasePriority       → 用於優先權繼承              │
├────────────────────────────────────────────────────┤
│ uxTCBNumber          → 唯一任務識別碼              │
├────────────────────────────────────────────────────┤
│ ...（其他可選欄位）...                             │
└────────────────────────────────────────────────────┘
```

### 3.2 堆疊佈局（ARM Cortex-M4F）
```
初始化後的堆疊記憶體佈局（堆疊向下增長 ↓）：

高位記憶體
┌──────────────────────────────────────┐
│                                      │
│         未使用的堆疊空間             │  ← pxStack（堆疊基底）
│                                      │
├──────────────────────────────────────┤
│          0xa5a5a5a5 (填充)           │  堆疊填充 0xa5 用於除錯
│          0xa5a5a5a5 (填充)           │
│          0xa5a5a5a5 (填充)           │
├──────────────────────────────────────┤
│  R4                                  │  ← 軟體保存的暫存器
│  R5                                  │     (由 PendSV 保存/恢復)
│  R6                                  │
│  R7                                  │
│  R8                                  │
│  R9                                  │
│  R10                                 │
│  R11                                 │
├──────────────────────────────────────┤
│  EXC_RETURN (0xFFFFFFFD)             │  ← 異常返回值
├──────────────────────────────────────┤
│  R0 = pvParameters                   │  ← 硬體保存的暫存器
│  R1 = 未初始化                       │     (由異常機制保存/恢復)
│  R2 = 未初始化                       │
│  R3 = 未初始化                       │
│  R12 = 未初始化                      │
├──────────────────────────────────────┤
│  LR = portTASK_RETURN_ADDRESS        │  ← 連結暫存器（退出函數）
├──────────────────────────────────────┤
│  PC = pxTaskCode (任務函數)          │  ← 程式計數器（進入點）
├──────────────────────────────────────┤
│  xPSR = 0x01000000 (T 位元設定)      │  ← 程式狀態暫存器
├──────────────────────────────────────┤  ← pxTopOfStack 指向這裡
低位記憶體

注意：此佈局模擬任務被中斷時的暫存器狀態，
      準備好由排程器恢復。
```

### 3.3 就緒列表結構
```
就緒任務列表（基於優先權）：

pxReadyTasksLists[configMAX_PRIORITIES]：
┌─────────────────────────────────────────────┐
│ 優先權 0（閒置）：List_t                    │
│   ├─→ 任務 A (StateListItem) ←──────┐      │
│   └─→ 任務 B (StateListItem)        │      │
├─────────────────────────────────────────────┤
│ 優先權 1：List_t                            │
│   └─→（空）                                 │
├─────────────────────────────────────────────┤
│ 優先權 2：List_t                            │
│   └─→ 任務 C (StateListItem)               │
├─────────────────────────────────────────────┤
│ 優先權 3：List_t                            │
│   ├─→ 任務 D (StateListItem)               │
│   └─→ 任務 E (StateListItem)               │
├─────────────────────────────────────────────┤
│  ...                                        │
├─────────────────────────────────────────────┤
│ 優先權 (configMAX_PRIORITIES-1)：List_t    │
│   └─→ 任務 F (StateListItem)               │
└─────────────────────────────────────────────┘

每個任務的 StateListItem 連結回其 TCB：
ListItem_t.pvOwner → TCB_t

插入：任務插入其優先權列表的末端
      使用 listINSERT_END() 實現相同優先權任務間的
      輪轉排程。
```

### 3.4 完整任務創建流程圖
```
┌─────────────────────────────────────────────────────────────────┐
│                     使用者應用程式                              │
│                  xTaskCreate(...)                               │
└──────────────────────┬──────────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              步驟 1：分配記憶體                                 │
│              prvCreateTask()                                    │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ 1a. 分配堆疊記憶體                                     │    │
│  │     pxStack = pvPortMallocStack(size)                  │    │
│  │                                                         │    │
│  │ 1b. 分配 TCB                                           │    │
│  │     pxNewTCB = pvPortMalloc(sizeof(TCB_t))             │    │
│  │                                                         │    │
│  │ 1c. 將 TCB 清零初始化                                  │    │
│  │     memset(pxNewTCB, 0, sizeof(TCB_t))                 │    │
│  │                                                         │    │
│  │ 1d. 連結堆疊至 TCB                                     │    │
│  │     pxNewTCB->pxStack = pxStack                        │    │
│  └────────────────────────────────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              步驟 2：初始化 TCB 欄位                            │
│              prvInitialiseNewTask()                             │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ 2a. 用 0xa5 填充堆疊（除錯用）                         │    │
│  │                                                         │    │
│  │ 2b. 計算堆疊頂端                                       │    │
│  │     pxTopOfStack = &pxStack[uxStackDepth-1]            │    │
│  │     （含對齊處理）                                     │    │
│  │                                                         │    │
│  │ 2c. 儲存任務名稱                                       │    │
│  │     strcpy(pxNewTCB->pcTaskName, pcName)               │    │
│  │                                                         │    │
│  │ 2d. 設定優先權                                         │    │
│  │     pxNewTCB->uxPriority = uxPriority                  │    │
│  │     pxNewTCB->uxBasePriority = uxPriority              │    │
│  │                                                         │    │
│  │ 2e. 初始化列表項                                       │    │
│  │     vListInitialiseItem(&StateListItem)                │    │
│  │     vListInitialiseItem(&EventListItem)                │    │
│  │     listSET_LIST_ITEM_OWNER(..., pxNewTCB)             │    │
│  │                                                         │    │
│  │ 2f. 設定任務運行狀態                                   │    │
│  │     pxNewTCB->xTaskRunState = taskTASK_NOT_RUNNING     │    │
│  └────────────────────────────────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              步驟 3：初始化堆疊框架                             │
│              pxPortInitialiseStack() [ARM CM4F]                 │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ 3a. 推入 xPSR (0x01000000)                             │    │
│  │ 3b. 推入 PC（任務進入點）                              │    │
│  │ 3c. 推入 LR（返回位址）                                │    │
│  │ 3d. 推入 R0-R3, R12（R0 = pvParameters）               │    │
│  │ 3e. 推入 EXC_RETURN (0xFFFFFFFD)                       │    │
│  │ 3f. 推入 R4-R11（軟體情境）                            │    │
│  │                                                         │    │
│  │ 返回：pxTopOfStack（已更新）                           │    │
│  │ 儲存：pxNewTCB->pxTopOfStack = pxTopOfStack            │    │
│  └────────────────────────────────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│              步驟 4：加入就緒列表                               │
│              prvAddNewTaskToReadyList()                         │
│  ┌────────────────────────────────────────────────────────┐    │
│  │ 4a. 進入臨界區                                         │    │
│  │     taskENTER_CRITICAL()                               │    │
│  │                                                         │    │
│  │ 4b. 遞增任務計數器                                     │    │
│  │     uxCurrentNumberOfTasks++                           │    │
│  │                                                         │    │
│  │ 4c. 第一個任務的初始化                                 │    │
│  │     if (pxCurrentTCB == NULL)                          │    │
│  │         pxCurrentTCB = pxNewTCB                         │    │
│  │         if (第一個任務) prvInitialiseTaskLists()       │    │
│  │                                                         │    │
│  │ 4d. 若優先權較高則更新 pxCurrentTCB                    │    │
│  │     （僅在排程器未運行時）                             │    │
│  │                                                         │    │
│  │ 4e. 分配 TCB 編號                                      │    │
│  │     pxNewTCB->uxTCBNumber = ++uxTaskNumber             │    │
│  │                                                         │    │
│  │ 4f. 插入就緒列表（巨集展開）：                         │    │
│  │     taskRECORD_READY_PRIORITY(priority)                │    │
│  │     listINSERT_END(                                    │    │
│  │         &pxReadyTasksLists[uxPriority],                │    │
│  │         &pxNewTCB->xStateListItem)                     │    │
│  │                                                         │    │
│  │ 4g. 離開臨界區                                         │    │
│  │     taskEXIT_CRITICAL()                                │    │
│  │                                                         │    │
│  │ 4h. 必要時讓出 CPU                                     │    │
│  │     if (排程器運行中 && 優先權較高)                    │    │
│  │         taskYIELD_IF_USING_PREEMPTION()                │    │
│  └────────────────────────────────────────────────────────┘    │
└──────────────────────┬──────────────────────────────────────────┘
                       ▼
              ┌────────────────────┐
              │  任務現在處於      │
              │  就緒狀態並準備    │
              │  進行排程          │
              └────────────────────┘
```

---

## 4. 總結

### 對 FreeRTOS 任務創建過程的理解

FreeRTOS 實現了一個系統化且結構良好的任務創建方法，確保任務被正確初始化並準備好進行排程。整個過程可以總結為四個主要階段：

**階段 1：記憶體分配**  
當 `xTaskCreate()` 被調用時，FreeRTOS 首先為任務控制塊（TCB）和任務堆疊分配記憶體。分配的順序取決於堆疊增長方向——對於 ARM Cortex-M4（堆疊向下增長），先分配堆疊，然後分配 TCB。這防止了堆疊可能破壞 TCB 結構。TCB 會被清零初始化，以確保所有欄位都從已知狀態開始。

**階段 2：TCB 初始化**  
`prvInitialiseNewTask()` 函數填充 TCB 的必要任務資訊。包括複製任務名稱、設定優先權（當前優先權和基礎優先權，用於優先權繼承），以及初始化兩個關鍵的列表項：`xStateListItem`（將任務連結到就緒/阻塞/暫停列表）和 `xEventListItem`（將任務連結到同步物件等待列表）。列表項配置為由 TCB 擁有，使 FreeRTOS 能夠快速從列表項導航回其包含的任務。

**階段 3：堆疊框架準備**  
架構特定的 `pxPortInitialiseStack()` 函數創建初始堆疊框架，模擬任務被中斷時的狀態。對於 ARM Cortex-M4F，這涉及在堆疊上放置 xPSR 暫存器（設定 Thumb 位元）、程式計數器（指向任務函數）、連結暫存器、通用暫存器（R0 包含任務參數）以及異常返回值。這個預初始化的堆疊允許排程器在第一次情境切換時「恢復」任務的情境，有效地在指定的進入點開始任務執行。

**階段 4：就緒列表插入**  
最後，`prvAddNewTaskToReadyList()` 在臨界區內將任務加入適當的就緒列表，以防止競態條件。FreeRTOS 維護一個就緒列表陣列——每個優先權等級一個。任務的 `xStateListItem` 使用 `listINSERT_END()` 插入其優先權特定列表的末端，這確保了相同優先權任務之間的輪轉排程。如果這是第一個任務，或者新任務的優先權高於當前任務（且排程器正在運行），FreeRTOS 可能會觸發情境切換以立即運行新任務。

**關鍵設計洞察：**  
1. TCB 的第一個欄位必須始終是 `pxTopOfStack`，以實現高效的組合語言級情境切換
2. 臨界區保護所有對共享資料結構（任務列表）的修改
3. 雙列表項設計（狀態和事件）允許任務同時存在於就緒/阻塞列表和等待同步物件
4. 優先權排序的事件列表（使用反轉的優先權值）能快速選擇最高優先權的等待任務
5. 堆疊預先初始化為已知模式（0xa5）有助於堆疊溢出檢測

這個架構展示了 FreeRTOS 在可攜性（`tasks.c` 中的通用任務管理）和硬體最佳化（移植檔案中的架構特定堆疊初始化）之間的仔細平衡。
