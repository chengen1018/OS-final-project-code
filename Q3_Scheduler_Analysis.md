# Q3: 調度器任務選擇與 TCB 調度欄位分析

## 第一部分：代碼追蹤流程（函數調用序列）

### 情境 1：透過 PendSV 中斷進行上下文切換（搶占式調度）

```
1. xPortPendSVHandler()                 [port.c:505]
   │   (組合語言程序 - 保存當前任務上下文)
   │
   ├─→ 將當前任務暫存器保存到堆疊 (R4-R11, R14)
   ├─→ 使用當前堆疊指標更新 pxCurrentTCB->pxTopOfStack
   │
   ├─→ 2. vTaskSwitchContext()          [tasks.c:5056]
   │      │
   │      ├─→ 檢查調度器是否暫停 (uxSchedulerSuspended)
   │      │   ├─→ 如果暫停：設定 xYieldPendings[0] = pdTRUE，返回
   │      │   └─→ 如果未暫停：繼續
   │      │
   │      ├─→ 更新執行時間統計（如果啟用）
   │      ├─→ 檢查堆疊溢出
   │      ├─→ 保存當前任務的 errno
   │      │
   │      ├─→ 3. taskSELECT_HIGHEST_PRIORITY_TASK()  [tasks.c:178-193]
   │      │      │
   │      │      ├─→ 從 uxTopReadyPriority 開始
   │      │      │
   │      │      ├─→ 4. 搜尋最高優先級的非空就緒列表
   │      │      │      while (listLIST_IS_EMPTY(&pxReadyTasksLists[uxTopPriority]))
   │      │      │          uxTopPriority--;  // 向下搜尋直到找到非空列表
   │      │      │
   │      │      └─→ 5. listGET_OWNER_OF_NEXT_ENTRY()  [list.h:286-297]
   │      │             │
   │      │             ├─→ 將 pxIndex 指向就緒列表的下一個項目
   │      │             ├─→ 如果遇到 xListEnd 標記則跳過
   │      │             ├─→ 從列表項目取得 pvOwner（TCB 指標）
   │      │             └─→ 將 pxCurrentTCB 設定為選中的 TCB
   │      │
   │      ├─→ 從新任務更新全域 errno
   │      └─→ 呼叫 portTASK_SWITCH_HOOK（如果有定義）
   │
   └─→ 從 pxCurrentTCB->pxTopOfStack 恢復新任務上下文
       └─→ 返回新任務執行
```

### 情境 2：時間片到期（輪轉調度）

```
1. xPortSysTickHandler()                [port.c:562]
   │   (SysTick 中斷處理器)
   │
   ├─→ 2. xTaskIncrementTick()          [tasks.c]
   │      ├─→ 遞增 xTickCount
   │      ├─→ 檢查需要解除阻塞的任務
   │      └─→ 如果需要上下文切換則返回 pdTRUE
   │
   ├─→ 如果需要上下文切換：
   │      ├─→ 設定 PendSV 中斷待處理
   │      │   portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT
   │      │
   │      └─→ (觸發 PendSV 處理器 - 參見情境 1)
   │
   └─→ 從中斷返回
```

### 情境 3：任務主動讓出 CPU

```
1. taskYIELD()                          [應用程式碼]
   │
   ├─→ portYIELD()                      [portmacro.h]
   │      └─→ 設定 PendSV 中斷待處理
   │
   └─→ (觸發 PendSV 處理器 - 參見情境 1)
```

---

## 第二部分：關鍵代碼片段（含文件名和函數名）

### 2.1 TCB 結構定義（調度相關欄位）

**文件：** `FreeRTOS/FreeRTOS-Kernel/tasks.c` (第 358-436 行)

```c
typedef struct tskTaskControlBlock
{
    // 關鍵：必須是第一個成員 - 硬體堆疊指標
    volatile StackType_t * pxTopOfStack;  /**< 指向堆疊上最後一個項目 */
    
    #if ( configUSE_CORE_AFFINITY == 1 ) && ( configNUMBER_OF_CORES > 1 )
        UBaseType_t uxCoreAffinityMask;   /**< SMP 系統的核心親和性 */
    #endif

    // *** 主要調度欄位 ***
    ListItem_t xStateListItem;            /**< 將任務連結到就緒/阻塞/暫停列表 */
    ListItem_t xEventListItem;            /**< 將任務連結到事件列表（佇列、信號量）*/
    UBaseType_t uxPriority;               /**< 當前優先級（0 = 最低）*/
    
    StackType_t * pxStack;                /**< 指向堆疊起始位置 */
    
    #if ( configNUMBER_OF_CORES > 1 )
        volatile BaseType_t xTaskRunState; /**< 核心 ID 或非執行狀態 */
        UBaseType_t uxTaskAttributes;      /**< 任務屬性（例如：閒置任務標記）*/
    #endif
    
    char pcTaskName[ configMAX_TASK_NAME_LEN ]; /**< 除錯用名稱 */

    #if ( configUSE_TASK_PREEMPTION_DISABLE == 1 )
        BaseType_t xPreemptionDisable;     /**< 設定時防止搶占 */
    #endif

    #if ( configUSE_MUTEXES == 1 )
        UBaseType_t uxBasePriority;        /**< 繼承前的原始優先級 */
        UBaseType_t uxMutexesHeld;         /**< 持有的互斥鎖數量 */
    #endif

    #if ( configGENERATE_RUN_TIME_STATS == 1 )
        configRUN_TIME_COUNTER_TYPE ulRunTimeCounter; /**< 使用的 CPU 時間 */
    #endif

    // ... 其他欄位用於通知、TLS 等
} tskTCB;
typedef tskTCB TCB_t;
```

### 2.2 就緒任務列表（基於優先級的存儲）

**文件：** `FreeRTOS/FreeRTOS-Kernel/tasks.c` (第 459-464 行)

```c
// 列表陣列 - 每個優先級一個
PRIVILEGED_DATA static List_t pxReadyTasksLists[ configMAX_PRIORITIES ];

// 其他任務狀態列表
PRIVILEGED_DATA static List_t xDelayedTaskList1;        // 延遲任務
PRIVILEGED_DATA static List_t xDelayedTaskList2;        // 溢出延遲任務
PRIVILEGED_DATA static List_t * volatile pxDelayedTaskList;
PRIVILEGED_DATA static List_t * volatile pxOverflowDelayedTaskList;
PRIVILEGED_DATA static List_t xPendingReadyList;        // 暫停期間變為就緒的任務

// 全域調度器變數
PRIVILEGED_DATA static volatile UBaseType_t uxTopReadyPriority = tskIDLE_PRIORITY;
PRIVILEGED_DATA static volatile BaseType_t xSchedulerRunning = pdFALSE;
```

### 2.3 任務選擇巨集（通用實作）

**文件：** `FreeRTOS/FreeRTOS-Kernel/tasks.c` (第 178-193 行)

```c
#define taskSELECT_HIGHEST_PRIORITY_TASK()                                       \
do {                                                                             \
    UBaseType_t uxTopPriority = uxTopReadyPriority;                              \
                                                                                 \
    /* 尋找包含就緒任務的最高優先級佇列 */                                        \
    while( listLIST_IS_EMPTY( &( pxReadyTasksLists[ uxTopPriority ] ) ) != pdFALSE ) \
    {                                                                            \
        configASSERT( uxTopPriority );                                           \
        --uxTopPriority;  /* 遞減直到找到非空列表 */                              \
    }                                                                            \
                                                                                 \
    /* listGET_OWNER_OF_NEXT_ENTRY 會遍歷列表，讓相同優先級的任務                \
     * 平均分享處理器時間 */                                                       \
    listGET_OWNER_OF_NEXT_ENTRY( pxCurrentTCB, &( pxReadyTasksLists[ uxTopPriority ] ) ); \
    uxTopReadyPriority = uxTopPriority;                                          \
} while( 0 )
```

### 2.4 透過列表遍歷實現輪轉調度

**文件：** `FreeRTOS/FreeRTOS-Kernel/include/list.h` (第 286-297 行)

```c
#define listGET_OWNER_OF_NEXT_ENTRY( pxTCB, pxList )                                       \
do {                                                                                       \
    List_t * const pxConstList = ( pxList );                                               \
    /* 將索引遞增到下一個項目並返回該項目，確保                                             \
     * 我們不會返回列表末尾使用的標記 */                                                    \
    ( pxConstList )->pxIndex = ( pxConstList )->pxIndex->pxNext;                           \
    if( ( void * ) ( pxConstList )->pxIndex == ( void * ) &( ( pxConstList )->xListEnd ) ) \
    {                                                                                      \
        ( pxConstList )->pxIndex = ( pxConstList )->xListEnd.pxNext; /* 回繞到開始 */      \
    }                                                                                      \
    ( pxTCB ) = ( pxConstList )->pxIndex->pvOwner;  /* 從列表項目取得 TCB */               \
} while( 0 )
```

### 2.5 上下文切換函數

**文件：** `FreeRTOS/FreeRTOS-Kernel/tasks.c` (第 5056-5139 行)

```c
void vTaskSwitchContext( void )
{
    traceENTER_vTaskSwitchContext();

    if( uxSchedulerSuspended != ( UBaseType_t ) 0U )
    {
        /* 調度器目前已暫停 - 不允許上下文切換 */
        xYieldPendings[ 0 ] = pdTRUE;
    }
    else
    {
        xYieldPendings[ 0 ] = pdFALSE;
        traceTASK_SWITCHED_OUT();

        #if ( configGENERATE_RUN_TIME_STATS == 1 )
        {
            // 更新當前任務的執行時間統計
            ulTotalRunTime[ 0 ] = portGET_RUN_TIME_COUNTER_VALUE();
            if( ulTotalRunTime[ 0 ] > ulTaskSwitchedInTime[ 0 ] )
            {
                pxCurrentTCB->ulRunTimeCounter += 
                    ( ulTotalRunTime[ 0 ] - ulTaskSwitchedInTime[ 0 ] );
            }
            ulTaskSwitchedInTime[ 0 ] = ulTotalRunTime[ 0 ];
        }
        #endif

        /* 如果有配置，檢查堆疊溢出 */
        taskCHECK_FOR_STACK_OVERFLOW();

        /* 在切換出當前執行的任務之前，保存其 errno */
        #if ( configUSE_POSIX_ERRNO == 1 )
        {
            pxCurrentTCB->iTaskErrno = FreeRTOS_errno;
        }
        #endif

        /* 使用通用 C 或移植優化的組合語言代碼選擇新任務執行 */
        taskSELECT_HIGHEST_PRIORITY_TASK();  // *** 關鍵調度器決策 ***
        traceTASK_SWITCHED_IN();

        /* 巨集，在切換任務後立即注入移植特定行為，
         * 例如設定堆疊溢出監視點或重新配置 MPU */
        portTASK_SWITCH_HOOK( pxCurrentTCB );

        /* 切換到新任務後，更新全域 errno */
        #if ( configUSE_POSIX_ERRNO == 1 )
        {
            FreeRTOS_errno = pxCurrentTCB->iTaskErrno;
        }
        #endif
    }

    traceRETURN_vTaskSwitchContext();
}
```

### 2.6 PendSV 處理器（硬體上下文切換）

**文件：** `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c` (第 505-559 行)

```c
void xPortPendSVHandler( void )
{
    __asm volatile
    (
        "   mrs r0, psp                         \n"  // 取得行程堆疊指標
        "   isb                                 \n"
        "                                       \n"
        "   ldr r3, pxCurrentTCBConst           \n"  // 取得 &pxCurrentTCB
        "   ldr r2, [r3]                        \n"  // 取得 pxCurrentTCB 值
        "                                       \n"
        "   tst r14, #0x10                      \n"  // 檢查是否使用 FPU
        "   it eq                               \n"
        "   vstmdbeq r0!, {s16-s31}             \n"  // 如果使用則保存 FPU 暫存器
        "                                       \n"
        "   stmdb r0!, {r4-r11, r14}            \n"  // 保存核心暫存器
        "   str r0, [r2]                        \n"  // 將堆疊指標保存到 TCB
        "                                       \n"
        "   stmdb sp!, {r0, r3}                 \n"
        "   mov r0, %0                          \n"
        "   msr basepri, r0                     \n"
        "   dsb                                 \n"
        "   isb                                 \n"
        "   bl vTaskSwitchContext               \n"  // *** 呼叫調度器 ***
        "   mov r0, #0                          \n"
        "   msr basepri, r0                     \n"
        "   ldmia sp!, {r0, r3}                 \n"
        "                                       \n"
        "   ldr r1, [r3]                        \n"  // 取得新的 pxCurrentTCB
        "   ldr r0, [r1]                        \n"  // 取得新的堆疊指標
        "                                       \n"
        "   ldmia r0!, {r4-r11, r14}            \n"  // 恢復核心暫存器
        "                                       \n"
        "   tst r14, #0x10                      \n"
        "   it eq                               \n"
        "   vldmiaeq r0!, {s16-s31}             \n"  // 如果使用則恢復 FPU
        "                                       \n"
        "   msr psp, r0                         \n"  // 設定新的堆疊指標
        "   isb                                 \n"
        "   bx r14                              \n"  // 返回新任務
        "                                       \n"
        "   .align 4                            \n"
        "pxCurrentTCBConst: .word pxCurrentTCB  \n"
        ::"i" ( configMAX_SYSCALL_INTERRUPT_PRIORITY )
    );
}
```

---

## 第三部分：解釋性圖表

### 圖表 1：TCB 與就緒列表結構

```
┌─────────────────────────────────────────────────────────────────────┐
│                     FreeRTOS 就緒列表結構                            │
└─────────────────────────────────────────────────────────────────────┘

pxReadyTasksLists[configMAX_PRIORITIES]        全域變數
┌──────────────────────────────┐              ┌──────────────────────┐
│ [優先級 4] List_t            │              │ uxTopReadyPriority=4 │
│  ├─ xListEnd (標記)          │              │ pxCurrentTCB ───┐    │
│  ├─ pxIndex ──┐              │              └─────────────────┼────┘
│  └─ uxNumberOfItems = 2      │                                │
│                               │                                │
│      ┌───────────────────────┼────────┐                       │
│      │   ListItem             │        │                       │
│      │   (在 TCB_A 中)        ↓        ↓                       │
│      │   ┌──────────────────────────────────┐                 │
│      │   │ xItemValue = 優先級              │                 │
│  ┌───┼───│ pxNext ────────────┐             │                 │
│  │   │   │ pxPrevious ────────┼─────┐       │                 │
│  │   └───│ pvOwner ─────────► TCB_A │       │                 │
│  │       │ pxContainer ───► List_t  │       │                 │
│  │       └──────────────────────────┼───────┘                 │
│  │                                  │                         │
│  │       ┌──────────────────────────┼───────┐                 │
│  │       │ xItemValue = 優先級      ↓       │                 │
│  │   ┌───│ pxNext ────────────┐             │                 │
│  └───┼───│ pxPrevious ────────┼────────┐    │                 │
│      └───│ pvOwner ─────────► TCB_B ◄─┼────┼─────────────────┘
│          │ pxContainer ───► List_t    │    │
│          └───────────────────────────┼┘    │
│                                      │     │
└──────────────────────────────────────┼─────┼───────────────────────┐
│ [優先級 3] List_t                    ↓     ↓                        │
│  ├─ xListEnd                    TCB_A 結構                          │
│  ├─ pxIndex                     ┌──────────────────────────────┐   │
│  └─ uxNumberOfItems = 1         │ pxTopOfStack ─► 堆疊記憶體   │   │
│      │                           │ xStateListItem (如上)        │   │
│      └──► [類似結構]             │ xEventListItem               │   │
└─────────────────────────────────│ uxPriority = 4               │───┘
┌────────────────────────────────────│ pxStack                      │
│ [優先級 2] List_t              │ xTaskRunState = RUNNING      │
│  ... (空的)                     │ uxTaskAttributes            │
└──────────────────────────────────│ pcTaskName = "Task_A"       │
┌────────────────────────────────────│ uxBasePriority (如果有互斥鎖)│
│ [優先級 1] List_t              │ ulRunTimeCounter (如果統計) │
│  ... (空的)                     └─────────────────────────────┘
└──────────────────────────────────
┌────────────────────────────────────
│ [優先級 0] List_t (閒置)
│  ├─ 包含閒置任務
│  └─ 至少總是有一個任務
└──────────────────────────────────
```

### 圖表 2：任務選擇演算法流程

```
┌─────────────────────────────────────────────────────────────────────┐
│              調度器任務選擇演算法（單核心）                          │
└─────────────────────────────────────────────────────────────────────┘

開始：vTaskSwitchContext() 被呼叫
  │
  ├─► 檢查：uxSchedulerSuspended != 0?
  │      是 ──► 設定 xYieldPendings[0] = TRUE ──► 返回（不切換）
  │      否 ──► 繼續
  │
  ├─► 更新當前任務的執行時間統計
  │   (ulRunTimeCounter += 經過時間)
  │
  ├─► taskCHECK_FOR_STACK_OVERFLOW()
  │
  ├─► 保存當前任務的 errno（如果啟用 POSIX）
  │
  ├─► taskSELECT_HIGHEST_PRIORITY_TASK()
  │     │
  │     ├─► uxTopPriority = uxTopReadyPriority（從最高開始）
  │     │
  │     ├─► 迴圈：while (pxReadyTasksLists[uxTopPriority] 是空的)
  │     │      │
  │     │      ├─► uxTopPriority--  (搜尋較低優先級)
  │     │      │
  │     │      └─► ASSERT：uxTopPriority > 0（必須有閒置任務）
  │     │
  │     ├─► 在優先級 uxTopPriority 找到非空列表
  │     │
  │     ├─► listGET_OWNER_OF_NEXT_ENTRY(pxCurrentTCB, &pxReadyTasksLists[uxTopPriority])
  │     │     │
  │     │     │   // 這為相同優先級的任務實現輪轉調度
  │     │     │
  │     │     ├─► pxList->pxIndex = pxList->pxIndex->pxNext（前進到下一個）
  │     │     │
  │     │     ├─► 如果到達 xListEnd 標記：
  │     │     │      pxList->pxIndex = xListEnd.pxNext（回繞到第一個任務）
  │     │     │
  │     │     └─► pxCurrentTCB = pxList->pxIndex->pvOwner（取得 TCB）
  │     │
  │     └─► 更新 uxTopReadyPriority = uxTopPriority
  │
  ├─► traceTASK_SWITCHED_IN()
  │
  ├─► portTASK_SWITCH_HOOK(pxCurrentTCB)（如果有定義）
  │
  ├─► 恢復新任務的 errno（如果啟用 POSIX）
  │
  └─► 返回到 PendSV 處理器
        │
        └─► 硬體從 pxCurrentTCB->pxTopOfStack 恢復新任務上下文
            └─► 恢復所選任務的執行

┌─────────────────────────────────────────────────────────────────────┐
│  輪轉機制（相同優先級任務）                                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  時間 ──►                                                            │
│                                                                      │
│  優先級 2 列表：[TaskA] ◄─► [TaskB] ◄─► [TaskC] ◄─► (回繞)         │
│                      ▲           ▲           ▲                       │
│  上下文切換 1：      │           │           │                       │
│     pxIndex ─────────┘           │           │                       │
│     選擇：TaskA                  │           │                       │
│                                  │           │                       │
│  上下文切換 2：                  │           │                       │
│     pxIndex ─────────────────────┘           │                       │
│     選擇：TaskB                              │                       │
│                                              │                       │
│  上下文切換 3：                              │                       │
│     pxIndex ─────────────────────────────────┘                       │
│     選擇：TaskC                                                      │
│                                                                      │
│  上下文切換 4：                                                      │
│     pxIndex 回繞到 TaskA（循環）                                     │
│                                                                      │
│  結果：相同優先級任務間的公平時間分享                                │
└─────────────────────────────────────────────────────────────────────┘
```

### 圖表 3：基於優先級的調度範例

```
┌─────────────────────────────────────────────────────────────────────┐
│              基於優先級的搶占式調度範例                              │
└─────────────────────────────────────────────────────────────────────┘

時間 ──────────────────────────────────────────────────────────►

任務優先級：
  - TaskA：優先級 3（最高用戶任務）
  - TaskB：優先級 2
  - TaskC：優先級 1
  - Idle： 優先級 0（最低）

情境時間線：
┌───────────────────────────────────────────────────────────────────┐
│ t0    t1     t2      t3      t4      t5      t6      t7      t8   │
│ │     │      │       │       │       │       │       │       │    │
│ ├─────┼──────┼───────┼───────┼───────┼───────┼───────┼───────┤    │
│                                                                    │
│ TaskA │██████│       │███████████████│       │                    │
│       │      │       │               │       │                    │
│ TaskB │      │███████│               │███████│                    │
│       │      │       │               │       │                    │
│ TaskC │      │       │               │       │████████████████    │
│       │      │       │               │       │                    │
│ Idle  │      │       │               │       │                    │
└───────┴──────┴───────┴───────────────┴───────┴─────────────────────┘
│       │      │       │               │       │
│       │      │       │               │       │
事件：  │      │       │               │       │
        │      │       │               │       │
        │      │       │               │       └─► t6：TaskC 喚醒
        │      │       │               │           （優先級高於 Idle）
        │      │       │               │           ► 搶占 Idle
        │      │       │               │
        │      │       │               └─► t5：TaskB 因信號量阻塞
        │      │       │                   ► 選擇 TaskC（次高就緒任務）
        │      │       │
        │      │       └─► t3：TaskA 從延遲中喚醒
        │      │           （最高優先級就緒）
        │      │           ► 立即搶占 TaskB
        │      │
        │      └─► t2：TaskA 阻塞等待事件
        │          ► 選擇 TaskB（次高就緒任務）
        │
        └─► t1：TaskA 開始執行
            （系統中最高優先級任務）

調度器決策點：
─────────────────────────────
t1：初始調度
    └─► taskSELECT 發現優先級 3 列表非空 → 選擇 TaskA

t2：TaskA 阻塞（事件等待）
    └─► 移至 xSuspendedTaskList
    └─► taskSELECT 發現優先級 2 列表非空 → 選擇 TaskB

t3：TaskA 解除阻塞（接收到事件）
    └─► 移至 pxReadyTasksLists[3]
    └─► 搶占：TaskA 優先級(3) > TaskB 優先級(2)
    └─► 觸發 PendSV → taskSELECT 選擇 TaskA

t4：TaskA 時間片到期
    └─► 仍為最高優先級
    └─► 沒有其他優先級 3 任務 → TaskA 繼續

t5：TaskA 完成/阻塞
    └─► taskSELECT 發現優先級 2 列表非空 → 選擇 TaskB

t6：TaskC 解除阻塞
    └─► 移至 pxReadyTasksLists[1]
    └─► TaskB 仍較高 → 不搶占

t7：TaskB 阻塞
    └─► taskSELECT 發現優先級 1 列表非空 → 選擇 TaskC
```

### 圖表 4：上下文切換狀態機

```
┌─────────────────────────────────────────────────────────────────────┐
│                   上下文切換狀態轉換                                 │
└─────────────────────────────────────────────────────────────────────┘

                    執行中的任務 (TaskX)
                          │
                          │ 發生中斷：
                          │  - SysTick（時間片）
                          │  - PendSV（手動讓出）
                          │  - 事件（任務解除阻塞）
                          ↓
            ┌─────────────────────────────┐
            │   PendSV 異常進入            │
            │  (硬體保存 R0-R3,           │
            │   R12, LR, PC, xPSR)        │
            └──────────────┬──────────────┘
                           │
                           ↓
            ┌─────────────────────────────┐
            │  xPortPendSVHandler()       │
            │  軟體保存：                 │
            │   - R4-R11（核心暫存器）    │
            │   - R14 (EXC_RETURN)        │
            │   - S16-S31（若使用 FPU）   │
            │                             │
            │  pxCurrentTCB->pxTopOfStack │
            │         = SP                │
            └──────────────┬──────────────┘
                           │
                           ↓
            ┌─────────────────────────────┐
            │  vTaskSwitchContext()       │
            │  ┌─────────────────────┐    │
            │  │ 調度器邏輯：        │    │
            │  │                     │    │
            │  │ 1. 找到最高優先級   │    │
            │  │    就緒列表         │    │
            │  │                     │    │
            │  │ 2. 輪轉選擇該列表   │    │
            │  │    中的下一個任務   │    │
            │  │                     │    │
            │  │ 3. 更新             │    │
            │  │    pxCurrentTCB     │    │
            │  └─────────────────────┘    │
            └──────────────┬──────────────┘
                           │
                           ↓
            ┌─────────────────────────────┐
            │  xPortPendSVHandler()       │
            │  （繼續）                   │
            │  軟體恢復：                 │
            │   - 新任務的 R4-R11         │
            │   - S16-S31（若使用 FPU）   │
            │                             │
            │  SP = 新的 pxTopOfStack     │
            └──────────────┬──────────────┘
                           │
                           ↓
            ┌─────────────────────────────┐
            │  PendSV 異常返回             │
            │  (硬體恢復 R0-R3,           │
            │   R12, LR, PC, xPSR)        │
            └──────────────┬──────────────┘
                           │
                           ↓
                    執行中的任務 (TaskY)
                    （新選擇的任務）

┌─────────────────────────────────────────────────────────────────────┐
│                    上下文切換期間的堆疊布局                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  舊任務堆疊                       新任務堆疊                         │
│  （保存後）                       （恢復前）                         │
│                                                                      │
│  較高位址                         較高位址                           │
│  ┌──────────────┐                 ┌──────────────┐                  │
│  │  [任務資料]  │                 │  [任務資料]  │                  │
│  ├──────────────┤                 ├──────────────┤                  │
│  │     ...      │                 │     ...      │                  │
│  ├──────────────┤                 ├──────────────┤                  │
│  │  S31 (FPU)   │                 │  S31 (FPU)   │                  │
│  │  S30         │                 │  S30         │                  │
│  │   ...        │                 │   ...        │                  │
│  │  S16         │                 │  S16         │                  │
│  ├──────────────┤                 ├──────────────┤                  │
│  │  R14 (LR)    │                 │  R14 (LR)    │                  │
│  │  R11         │                 │  R11         │                  │
│  │  R10         │                 │  R10         │                  │
│  │   ...        │                 │   ...        │                  │
│  │  R4          │                 │  R4          │                  │
│  ├──────────────┤ ◄─ pxTopOfStack ├──────────────┤ ◄─ pxTopOfStack  │
│  │  xPSR        │    （保存在     │  xPSR        │    （從 TCB      │
│  │  PC          │     TCB 中）    │  PC          │     載入）       │
│  │  LR          │                 │  LR          │                  │
│  │  R12         │                 │  R12         │                  │
│  │  R3          │                 │  R3          │                  │
│  │  R2          │                 │  R2          │                  │
│  │  R1          │                 │  R1          │                  │
│  │  R0          │                 │  R0          │                  │
│  └──────────────┘                 └──────────────┘ ◄─ PSP           │
│  較低位址                         較低位址                           │
│                                                                      │
│  (硬體)     = 由 CPU 在異常期間保存/恢復                            │
│  (軟體)     = 由 PendSV 處理器保存/恢復                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 第四部分：控制調度行為的 TCB 欄位

### 4.1 主要調度欄位

| 欄位 | 類型 | 用途 | 對調度的影響 |
|------|------|------|-------------|
| **`pxTopOfStack`** | `volatile StackType_t*` | 指向任務堆疊的當前頂端 | **關鍵**：必須是 TCB 的第一個成員。硬體在上下文切換期間使用此欄位來保存/恢復任務狀態。 |
| **`xStateListItem`** | `ListItem_t` | 將任務連結到調度器列表（就緒/阻塞/暫停）| **主要**：決定哪個狀態列表包含該任務。`xItemValue` 通常保存優先級值用於排序。 |
| **`uxPriority`** | `UBaseType_t` | 當前任務優先級（0=最低，configMAX_PRIORITIES-1=最高）| **主要**：調度器始終選擇最高優先級的就緒任務。決定任務被插入到哪個 `pxReadyTasksLists[priority]`。 |

### 4.2 次要調度欄位

| 欄位 | 類型 | 用途 | 調度影響 |
|------|------|------|----------|
| **`xEventListItem`** | `ListItem_t` | 將任務連結到事件列表（佇列、信號量、延遲）| 當任務在同步物件上阻塞時，透過此項目移至事件列表。`xItemValue` 通常保存喚醒時間或優先級。 |
| **`xTaskRunState`** | `volatile BaseType_t` | （僅 SMP）指示哪個核心正在執行任務，或非執行狀態 | 防止同一任務在多個核心上執行。值：核心 ID 或 `taskTASK_NOT_RUNNING`。 |
| **`uxCoreAffinityMask`** | `UBaseType_t` | （僅 SMP）任務可以執行的核心位元遮罩 | 調度器僅為符合親和性遮罩的核心選擇任務。 |

### 4.3 優先級繼承欄位

| 欄位 | 類型 | 用途 | 調度影響 |
|------|------|------|----------|
| **`uxBasePriority`** | `UBaseType_t` | 繼承前的原始優先級 | 當任務持有較高優先級任務需要的互斥鎖時，`uxPriority` 會暫時提高。`uxBasePriority` 儲存原始值以便恢復。 |
| **`uxMutexesHeld`** | `UBaseType_t` | 當前持有的互斥鎖數量 | 防止在仍持有互斥鎖時降低優先級。 |

### 4.4 搶占控制欄位

| 欄位 | 類型 | 用途 | 調度影響 |
|------|------|------|----------|
| **`xPreemptionDisable`** | `BaseType_t` | （可選）禁用此任務的搶占 | 如果啟用，即使有較高優先級的任務變為就緒，任務也不能被搶占。少見的使用情況。 |
| **`uxTaskAttributes`** | `UBaseType_t` | （僅 SMP）標誌，如 `taskATTRIBUTE_IS_IDLE` | 閒置任務由此標誌識別；調度器特別對待它們（始終為最低有效優先級）。 |

### 4.5 監控/除錯欄位（無直接調度影響）

| 欄位 | 類型 | 用途 |
|------|------|------|
| **`ulRunTimeCounter`** | `configRUN_TIME_COUNTER_TYPE` | 累積任務使用的 CPU 時間 |
| **`pcTaskName`** | `char[]` | 用於任務識別的除錯名稱 |
| **`uxTCBNumber`** | `UBaseType_t` | 除錯器的唯一 ID |

---

## 第五部分：總結段落

**FreeRTOS 實現了一個**基於優先級的搶占式調度器**，並在相同優先級的任務之間採用**輪轉時間分享**機制。調度器維護一個就緒列表陣列 (`pxReadyTasksLists[]`)，每個優先級一個列表，任務透過其 TCB 中的 `xStateListItem` 欄位連結到這些列表。當發生上下文切換時（由 SysTick 計時器、手動讓出或任務狀態變更觸發），在 PendSV 異常處理器內呼叫 `vTaskSwitchContext()` 函數。調度器從快取的 `uxTopReadyPriority` 向下搜尋，使用 `taskSELECT_HIGHEST_PRIORITY_TASK()` 找到最高優先級的非空就緒列表。然後透過 `listGET_OWNER_OF_NEXT_ENTRY()` 選擇該列表中的下一個任務，該函數會循環前進列表的 `pxIndex` 指標，實現相同優先級任務的輪轉調度。所選任務的 TCB 位址成為新的 `pxCurrentTCB`，PendSV 處理器從 `pxCurrentTCB->pxTopOfStack` 恢復其上下文。控制此行為的關鍵 TCB 欄位包括：`uxPriority`（決定就緒列表放置位置和選擇順序）、`xStateListItem`（將任務連結到就緒/阻塞列表）、`pxTopOfStack`（儲存硬體堆疊指標以進行上下文恢復），以及可選的 `uxBasePriority`/`uxMutexesHeld`（用於優先級繼承）。這種設計確保了對高優先級事件的確定性、低延遲響應，同時在相同優先級任務之間提供公平的 CPU 分享。**

---

## 關於優化的說明

1. **通用 vs. 移植優化選擇**：
   - 通用方法（如上所示）：使用線性搜尋優先級層級。
   - 移植優化方法 (`configUSE_PORT_OPTIMISED_TASK_SELECTION=1`)：使用硬體位元掃描指令（例如 ARM 上的 CLZ）實現 O(1) 優先級查找。

2. **單核心 vs. SMP**：
   - 單核心：簡單的 `pxCurrentTCB` 指標。
   - SMP：陣列 `pxCurrentTCBs[configNUMBER_OF_CORES]`，複雜的親和性和執行狀態管理。

3. **調度器暫停**：
   - 當 `uxSchedulerSuspended > 0` 時，防止上下文切換。
   - 不讓出的任務放置在 `xPendingReadyList` 中以便稍後處理。

---

## 參考資料

- **文件**：`FreeRTOS/FreeRTOS-Kernel/tasks.c`
  - 第 358-436 行：TCB 結構
  - 第 178-193 行：taskSELECT_HIGHEST_PRIORITY_TASK 巨集
  - 第 5056-5139 行：vTaskSwitchContext 函數

- **文件**：`FreeRTOS/FreeRTOS-Kernel/include/list.h`
  - 第 286-297 行：listGET_OWNER_OF_NEXT_ENTRY 巨集
  - 第 144-179 行：List 和 ListItem 結構

- **文件**：`FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c`
  - 第 505-559 行：xPortPendSVHandler（硬體上下文切換）
  - 第 562-586 行：xPortSysTickHandler（計時器 tick）

---

**文件建立日期**：2026-01-07  
**FreeRTOS 版本**：v11.1.0 LTS  
**分析架構**：ARM Cortex-M4F（單核心配置）
