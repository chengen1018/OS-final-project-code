# Q6: FreeRTOS Context Switch 實作分析

## 題目

追蹤 FreeRTOS 的 Context Switch 實作，並解釋：
1. 哪些暫存器由硬體儲存/還原，哪些由軟體處理
2. PendSV handler 如何與 vTaskSwitchContext() 互動
3. 新任務的堆疊指標如何在返回 Thread mode 之前被還原

---

# Part 1: 硬體與軟體的暫存器儲存/還原分工

## 1.1 程式碼追蹤流程 (Code Tracing Flow)

```
Context Switch 暫存器儲存流程：

1. 例外發生（如 PendSV 觸發）
   │
   ▼
2. [硬體自動執行] ARM Cortex-M Exception Entry
   │  自動將以下暫存器推入 PSP：
   │  R0, R1, R2, R3, R12, LR, PC, xPSR
   │  (若 FPU 啟用: S0-S15, FPSCR - Lazy Stacking)
   │
   ▼
3. [軟體執行] xPortPendSVHandler() 開始
   │  port.c 第 505 行
   │
   ├─▶ mrs r0, psp              (第 511 行) 取得當前 PSP
   ├─▶ ldr r2, [pxCurrentTCB]   (第 514-515 行) 取得 TCB
   ├─▶ tst r14, #0x10           (第 517 行) 檢查 FPU 使用
   ├─▶ vstmdbeq r0!, {s16-s31}  (第 519 行) 儲存 S16-S31 (若有用 FPU)
   ├─▶ stmdb r0!, {r4-r11, r14} (第 521 行) 儲存 R4-R11, EXC_RETURN
   └─▶ str r0, [r2]             (第 522 行) 更新 TCB->pxTopOfStack
```

## 1.2 關鍵程式碼片段 (Key Code Snippets)

**檔案：** `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c`  
**函式：** `xPortPendSVHandler()` (第 505-559 行)

```c
void xPortPendSVHandler( void )
{
    __asm volatile
    (
        // 硬體已自動儲存 R0-R3, R12, LR, PC, xPSR 到 PSP
        
        "   mrs r0, psp                         \n"  // 取得 PSP（硬體 stacking 後的位置）
        "   isb                                 \n"
        
        "   ldr r3, pxCurrentTCBConst           \n"  // R3 = &pxCurrentTCB
        "   ldr r2, [r3]                        \n"  // R2 = pxCurrentTCB
        
        // === 軟體儲存 FPU 高位暫存器（條件式）===
        "   tst r14, #0x10                      \n"  // 測試 bit 4
        "   it eq                               \n"  // 若 bit 4 = 0 (使用 FPU)
        "   vstmdbeq r0!, {s16-s31}             \n"  // 儲存 S16-S31
        
        // === 軟體儲存 R4-R11 和 EXC_RETURN ===
        "   stmdb r0!, {r4-r11, r14}            \n"  // 推入堆疊
        "   str r0, [r2]                        \n"  // 更新 TCB->pxTopOfStack
        // ...
    );
}
```

## 1.3 圖解說明 (Explanatory Diagram)

```
╔═══════════════════════════════════════════════════════════════════╗
║                    任務堆疊記憶體配置                              ║
╚═══════════════════════════════════════════════════════════════════╝

高位址
    │
    ├─────────────────┤ ◀── PSP（例外發生前）
    │                 │
    │  ╔═════════════╗│
    │  ║  xPSR       ║│
    │  ║  PC         ║│
    │  ║  LR         ║│      ┌────────────────────┐
    │  ║  R12        ║│ ◀────│ 硬體自動儲存       │
    │  ║  R3         ║│      │ (Exception Entry)  │
    │  ║  R2         ║│      └────────────────────┘
    │  ║  R1         ║│
    │  ║  R0         ║│
    │  ╚═════════════╝│
    ├─────────────────┤ ◀── PSP（硬體 stacking 後）
    │  (S0-S15)       │      Lazy FPU Stacking
    ├─────────────────┤
    │                 │
    │  ╔═════════════╗│
    │  ║  R11        ║│
    │  ║  R10        ║│
    │  ║  R9         ║│
    │  ║  R8         ║│      ┌────────────────────┐
    │  ║  R7         ║│ ◀────│ 軟體手動儲存       │
    │  ║  R6         ║│      │ (PendSV Handler)   │
    │  ║  R5         ║│      └────────────────────┘
    │  ║  R4         ║│
    │  ║  R14(EXC)   ║│
    │  ╚═════════════╝│
    ├─────────────────┤
    │  (S16-S31)      │      軟體儲存（若有使用 FPU）
    ├─────────────────┤ ◀── TCB->pxTopOfStack（最終值）
    │                 │
    ▼
低位址

╔═══════════════════════════════════════════════════════════════════╗
║                     暫存器儲存分工總表                             ║
╠═══════════════════╦═══════════════════╦═══════════════════════════╣
║   儲存方式        ║   暫存器          ║   說明                    ║
╠═══════════════════╬═══════════════════╬═══════════════════════════╣
║   硬體自動        ║ R0-R3, R12        ║   Caller-saved (volatile) ║
║   (Exception      ║ LR, PC, xPSR      ║   程式執行狀態            ║
║    Entry)         ║ S0-S15, FPSCR     ║   FPU (Lazy Stacking)     ║
╠═══════════════════╬═══════════════════╬═══════════════════════════╣
║   軟體手動        ║ R4-R11            ║   Callee-saved            ║
║   (PendSV         ║ R14 (EXC_RETURN)  ║   例外返回控制值          ║
║    Handler)       ║ S16-S31           ║   FPU 高位暫存器          ║
╚═══════════════════╩═══════════════════╩═══════════════════════════╝
```

## 1.4 小結 (Summary)

ARM Cortex-M4F 的暫存器儲存採用**硬體-軟體協作**機制：

1. **硬體負責 volatile 暫存器**：當例外發生時，處理器自動將 R0-R3、R12、LR、PC、xPSR 推入 PSP 堆疊。這些是 ARM 呼叫慣例中的 caller-saved 暫存器，函式呼叫時本就不保證保留。

2. **軟體負責 non-volatile 暫存器**：PendSV handler 以組合語言手動儲存 R4-R11（callee-saved）和 R14（EXC_RETURN）。這些暫存器必須跨函式呼叫保留，因此需要完整備份。

3. **FPU 採用 Lazy Stacking 優化**：S0-S15 由硬體延遲儲存，S16-S31 由軟體在確認使用 FPU 後才儲存，避免不必要的開銷。

---

# Part 2: PendSV Handler 與 vTaskSwitchContext() 的互動機制

## 2.1 程式碼追蹤流程 (Code Tracing Flow)

```
PendSV 與 vTaskSwitchContext 互動流程：

1. SysTick 中斷觸發
   └─▶ xPortSysTickHandler()                    [port.c:562]
       └─▶ xTaskIncrementTick()
           └─▶ 返回 pdTRUE（需要切換）
               └─▶ portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT
                   設定 NVIC 的 PendSV pending bit

2. 所有高優先權中斷處理完畢後，PendSV 例外觸發
   └─▶ xPortPendSVHandler()                     [port.c:505]
       │
       ├─▶ [第一階段：儲存舊任務 Context]       [port.c:511-522]
       │   mrs r0, psp
       │   stmdb r0!, {r4-r11, r14}
       │   str r0, [r2]  // 更新 TCB->pxTopOfStack
       │
       ├─▶ [第二階段：呼叫排程器]               [port.c:524-532]
       │   stmdb sp!, {r0, r3}
       │   msr basepri, MAX_PRI                 // 關閉中斷（臨界區開始）
       │   bl vTaskSwitchContext ─────────────────────┐
       │   msr basepri, #0                      //    │ 開啟中斷（臨界區結束）
       │   ldmia sp!, {r0, r3}                  //    │
       │                                        //    │
       │   ┌──────────────────────────────────────────┘
       │   ▼
       │   vTaskSwitchContext()                 [tasks.c:5056]
       │   ├─▶ 檢查 uxSchedulerSuspended
       │   ├─▶ traceTASK_SWITCHED_OUT()
       │   ├─▶ 更新執行時間統計
       │   ├─▶ taskCHECK_FOR_STACK_OVERFLOW()
       │   ├─▶ taskSELECT_HIGHEST_PRIORITY_TASK()
       │   │   └─▶ ★ 更新 pxCurrentTCB = 新任務的 TCB ★
       │   ├─▶ traceTASK_SWITCHED_IN()
       │   └─▶ 返回
       │
       └─▶ [第三階段：還原新任務 Context]       [port.c:534-553]
           ldr r1, [r3]  // 取得新 TCB（已更新）
           ldr r0, [r1]  // 取得新任務的 pxTopOfStack
           ldmia r0!, {r4-r11, r14}
           msr psp, r0
           bx r14        // 例外返回
```

## 2.2 關鍵程式碼片段 (Key Code Snippets)

### A. 觸發 PendSV

**檔案：** `portable/GCC/ARM_CM4F/portmacro.h` (第 88-100 行)

```c
#define portYIELD()                                     \
    {                                                   \
        portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT; \
        __asm volatile ( "dsb" ::: "memory" );          \
        __asm volatile ( "isb" );                       \
    }

#define portNVIC_INT_CTRL_REG     ( *( ( volatile uint32_t * ) 0xe000ed04 ) )
#define portNVIC_PENDSVSET_BIT    ( 1UL << 28UL )
```

### B. PendSV 呼叫排程器

**檔案：** `portable/GCC/ARM_CM4F/port.c` (第 524-532 行)

```c
        // === 進入臨界區並呼叫排程器 ===
        "   stmdb sp!, {r0, r3}                 \n"  // 保存到 MSP
        "   mov r0, %0                          \n"  // configMAX_SYSCALL_INTERRUPT_PRIORITY
        "   msr basepri, r0                     \n"  // 遮蔽中斷 ← 臨界區開始
        "   dsb                                 \n"
        "   isb                                 \n"
        "   bl vTaskSwitchContext               \n"  // 呼叫 C 函式
        "   mov r0, #0                          \n"
        "   msr basepri, r0                     \n"  // 解除遮蔽 ← 臨界區結束
        "   ldmia sp!, {r0, r3}                 \n"  // 從 MSP 還原
```

### C. vTaskSwitchContext 核心邏輯

**檔案：** `FreeRTOS-Kernel/tasks.c` (第 5056-5139 行)

```c
void vTaskSwitchContext( void )
{
    if( uxSchedulerSuspended != ( UBaseType_t ) 0U )
    {
        // 排程器暫停，延遲切換
        xYieldPendings[ 0 ] = pdTRUE;
    }
    else
    {
        xYieldPendings[ 0 ] = pdFALSE;
        traceTASK_SWITCHED_OUT();

        // 更新執行時間統計
        #if ( configGENERATE_RUN_TIME_STATS == 1 )
        {
            ulTotalRunTime[ 0 ] = portGET_RUN_TIME_COUNTER_VALUE();
            pxCurrentTCB->ulRunTimeCounter += 
                ( ulTotalRunTime[ 0 ] - ulTaskSwitchedInTime[ 0 ] );
            ulTaskSwitchedInTime[ 0 ] = ulTotalRunTime[ 0 ];
        }
        #endif

        // 檢查堆疊溢位
        taskCHECK_FOR_STACK_OVERFLOW();

        // ★★★ 核心：選擇最高優先權任務 ★★★
        taskSELECT_HIGHEST_PRIORITY_TASK();
        // 此巨集會更新 pxCurrentTCB 指向新任務
        
        traceTASK_SWITCHED_IN();
    }
}
```

## 2.3 圖解說明 (Explanatory Diagram)

```
╔═══════════════════════════════════════════════════════════════════╗
║              PendSV Handler 與 vTaskSwitchContext 互動架構        ║
╚═══════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────┐
│                    PendSV Handler（組合語言）                    │
│                    port.c 第 505-559 行                         │
│                                                                  │
│  ┌────────────────┐                                             │
│  │ 第一階段：儲存 │  mrs r0, psp                                │
│  │ 舊任務 Context │  stmdb r0!, {r4-r11, r14}                   │
│  │                │  str r0, [r2]                               │
│  └───────┬────────┘                                             │
│          │                                                       │
│          ▼                                                       │
│  ┌────────────────┐     ┌─────────────────────────────────┐     │
│  │ 第二階段：     │     │ vTaskSwitchContext()            │     │
│  │ 呼叫排程器     │     │ tasks.c 第 5056-5139 行         │     │
│  │                │     │                                 │     │
│  │ msr basepri,   │────▶│ • 檢查排程器狀態                │     │
│  │   MAX_PRI      │     │ • 更新執行統計                  │     │
│  │ bl vTaskSwitch │     │ • 堆疊溢位檢查                  │     │
│  │   Context      │     │ • taskSELECT_HIGHEST_PRIORITY   │     │
│  │ msr basepri, 0 │◀────│   _TASK()                       │     │
│  │                │     │   ↓                             │     │
│  └───────┬────────┘     │   更新 pxCurrentTCB             │     │
│          │               └─────────────────────────────────┘     │
│          ▼                                                       │
│  ┌────────────────┐                                             │
│  │ 第三階段：還原 │  ldr r1, [pxCurrentTCB]  // 新 TCB          │
│  │ 新任務 Context │  ldr r0, [r1]            // 新堆疊指標      │
│  │                │  ldmia r0!, {r4-r11, r14}                   │
│  │                │  msr psp, r0                                │
│  │                │  bx r14                                     │
│  └────────────────┘                                             │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

╔═══════════════════════════════════════════════════════════════════╗
║                         臨界區保護機制                            ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                    ║
║   msr basepri, configMAX_SYSCALL_INTERRUPT_PRIORITY               ║
║      │                                                             ║
║      ├────────────────────────────────────────┐                   ║
║      │         臨界區（中斷被遮蔽）           │                   ║
║      │                                        │                   ║
║      │    vTaskSwitchContext() 執行          │                   ║
║      │    pxCurrentTCB 更新                   │ ◀── 原子操作     ║
║      │                                        │                   ║
║      ├────────────────────────────────────────┘                   ║
║      │                                                             ║
║   msr basepri, #0                                                  ║
║                                                                    ║
╚═══════════════════════════════════════════════════════════════════╝
```

## 2.4 小結 (Summary)

PendSV handler 與 vTaskSwitchContext() 採用**兩層分離架構**：

1. **PendSV Handler（組合語言）**負責底層操作：
   - 直接存取 CPU 暫存器（mrs, msr, ldmia, stmdb）
   - 堆疊指標管理（PSP, MSP）
   - 例外返回控制（bx r14）

2. **vTaskSwitchContext()（C 語言）**負責高階邏輯：
   - 排程決策（taskSELECT_HIGHEST_PRIORITY_TASK）
   - 狀態管理（uxSchedulerSuspended 檢查）
   - 統計更新（執行時間、堆疊檢查）

3. **BASEPRI 臨界區保護**確保原子性：
   - 呼叫 vTaskSwitchContext 前遮蔽中斷
   - 保證 pxCurrentTCB 更新過程不被打斷
   - 返回後才解除遮蔽

這種設計的優點是**可移植性**（C 邏輯可跨平台）和**可維護性**（排程演算法修改不影響組合語言）。

---

# Part 3: 新任務堆疊指標的還原流程

## 3.1 程式碼追蹤流程 (Code Tracing Flow)

```
堆疊指標還原三階段流程：

vTaskSwitchContext() 返回後，pxCurrentTCB 已指向新任務

┌─────────────────────────────────────────────────────────────────┐
│ 階段一：載入新任務的堆疊指標                                     │
│ [port.c 第 534-535 行]                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    ldr r1, [r3]       ; r3 = &pxCurrentTCB                      │
│                       ; r1 = pxCurrentTCB（新任務的 TCB 位址）   │
│                                                                  │
│    ldr r0, [r1]       ; r0 = TCB->pxTopOfStack（堆疊頂端）       │
│                       ; 這是新任務上次被切出時保存的位置         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 階段二：還原軟體儲存的暫存器                                     │
│ [port.c 第 537-541 行]                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    ldmia r0!, {r4-r11, r14}                                     │
│         ; 彈出 9 個字組 (36 bytes)                               │
│         ; R4-R11: 還原到 CPU 暫存器                              │
│         ; R14: 還原 EXC_RETURN 值                                │
│         ; r0 自動遞增，指向下一區塊                              │
│                                                                  │
│    tst r14, #0x10     ; 測試 bit 4（FPU 使用標誌）               │
│    it eq              ; 若 bit 4 = 0                             │
│    vldmiaeq r0!, {s16-s31}  ; 還原 FPU 高位暫存器                │
│                       ; r0 再次遞增                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 階段三：設定 PSP 並觸發例外返回                                  │
│ [port.c 第 543-553 行]                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    msr psp, r0        ; 設定 Process Stack Pointer               │
│                       ; r0 現在指向硬體儲存的框架                │
│                       ; (R0-R3, R12, LR, PC, xPSR)               │
│                                                                  │
│    isb                ; 指令同步屏障                             │
│                       ; 確保 PSP 更新在 bx 前完成                │
│                                                                  │
│    bx r14             ; 跳轉到 EXC_RETURN (0xFFFFFFFD)           │
│                       ; ┌─────────────────────────────────┐     │
│                       ; │ 硬體自動執行：                   │     │
│                       ; │ • 從 PSP 彈出 R0-R3, R12, LR,   │     │
│                       ; │   PC, xPSR                       │     │
│                       ; │ • 若有 FPU 框架，彈出 S0-S15    │     │
│                       ; │ • 載入 PC → 新任務恢復執行      │     │
│                       ; │ • 切換到 Thread mode            │     │
│                       ; │ • 切換使用 PSP                  │     │
│                       ; └─────────────────────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 3.2 關鍵程式碼片段 (Key Code Snippets)

**檔案：** `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c`  
**函式：** `xPortPendSVHandler()` (第 534-558 行)

```c
        // === 階段一：載入新任務堆疊指標 ===
        "   ldr r1, [r3]                        \n"  // r1 = 新的 pxCurrentTCB
        "   ldr r0, [r1]                        \n"  // r0 = 新任務的 pxTopOfStack
        
        // === 階段二：還原軟體儲存的暫存器 ===
        "   ldmia r0!, {r4-r11, r14}            \n"  // 彈出 R4-R11, EXC_RETURN
        "                                       \n"
        "   tst r14, #0x10                      \n"  // 測試 FPU 使用
        "   it eq                               \n"
        "   vldmiaeq r0!, {s16-s31}             \n"  // 還原 S16-S31（若有）
        
        // === 階段三：設定 PSP 並例外返回 ===
        "   msr psp, r0                         \n"  // PSP = 新堆疊位置
        "   isb                                 \n"  // 同步屏障
        "                                       \n"
        "   bx r14                              \n"  // 例外返回
        "                                       \n"
        "   .align 4                            \n"
        "pxCurrentTCBConst: .word pxCurrentTCB  \n"
```

### EXC_RETURN 值說明

```c
// FreeRTOS 使用的 EXC_RETURN 值
#define portINITIAL_EXC_RETURN    ( 0xfffffffd )

// 位元解析：
// [31:4] = 0xFFFFFFF  → 例外返回標記
// [3]    = 1          → 返回 Thread mode（非 Handler mode）
// [2]    = 1          → 使用 PSP（非 MSP）
// [4]    = 1          → 標準框架 / Lazy FPU
```

## 3.3 圖解說明 (Explanatory Diagram)

```
╔═══════════════════════════════════════════════════════════════════╗
║                    堆疊指標還原過程示意圖                          ║
╚═══════════════════════════════════════════════════════════════════╝

pxCurrentTCB (已被 vTaskSwitchContext 更新)
     │
     ▼
┌─────────────────┐
│ TCB (新任務)    │
├─────────────────┤
│ pxTopOfStack ───┼──────┐
├─────────────────┤      │
│ xStateListItem  │      │
│ uxPriority      │      │
│ ...             │      │
└─────────────────┘      │
                          │
    新任務的堆疊記憶體     │
                          │
    ┌─────────────────┐   │
    │ R4              │◀──┘ ① ldr r0, [r1]
    │ R5              │       r0 = pxTopOfStack
    │ R6              │
    │ R7              │
    │ R8              │
    │ R9              │       ② ldmia r0!, {r4-r11, r14}
    │ R10             │          彈出到 CPU 暫存器
    │ R11             │          r0 自動 +36
    │ R14 (EXC_RET)   │
    ├─────────────────┤◀───── r0 指向這裡
    │ (S16-S31)       │       ③ vldmiaeq r0!, {s16-s31}
    │                 │          r0 自動 +64（若有 FPU）
    ├─────────────────┤◀───── r0 指向這裡
    │ R0              │
    │ R1              │
    │ R2              │       ④ msr psp, r0
    │ R3              │          設定 PSP 到這個位置
    │ R12             │
    │ LR              │       ⑤ bx r14
    │ PC              │ ◀─────── 硬體彈出，載入到 PC
    │ xPSR            │          新任務從這裡恢復執行
    ├─────────────────┤◀───── PSP（例外返回後）
    │ (S0-S15)        │
    └─────────────────┘


╔═══════════════════════════════════════════════════════════════════╗
║                         bx r14 觸發的硬體行為                      ║
╠═══════════════════════════════════════════════════════════════════╣
║                                                                    ║
║   當 CPU 執行 bx 0xFFFFFFFD 時：                                   ║
║                                                                    ║
║   1. 偵測到 EXC_RETURN 模式                                        ║
║   2. 從 PSP 彈出：R0, R1, R2, R3, R12, LR, PC, xPSR               ║
║   3. 若有 FPU 框架：彈出 S0-S15, FPSCR                             ║
║   4. 將 PC 載入彈出的值 → 新任務從該位址恢復執行                  ║
║   5. 處理器切換到 Thread mode                                      ║
║   6. 處理器切換使用 PSP（不再使用 MSP）                           ║
║                                                                    ║
╚═══════════════════════════════════════════════════════════════════╝
```

## 3.4 小結 (Summary)

新任務堆疊指標的還原是一個**三階段硬體-軟體協作**過程：

1. **階段一（軟體）**：從更新後的 pxCurrentTCB 讀取新任務的 TCB，再從 TCB 取得 pxTopOfStack。這個指標指向新任務上次被切出時保存的堆疊位置。

2. **階段二（軟體）**：使用 `ldmia` 指令從堆疊彈出軟體儲存的暫存器（R4-R11、EXC_RETURN、S16-S31），指標自動遞增到硬體儲存框架的起始位置。

3. **階段三（軟體觸發 → 硬體執行）**：
   - 軟體用 `msr psp, r0` 設定 PSP 指向硬體框架
   - 軟體執行 `bx r14`（r14 = 0xFFFFFFFD）
   - 硬體偵測到 EXC_RETURN 模式，自動從 PSP 彈出 R0-R3、R12、LR、PC、xPSR
   - PC 被載入新任務的返回位址，任務恢復執行
   - 處理器切換到 Thread mode，開始使用 PSP

這個設計充分利用了 ARM Cortex-M 的**例外返回機制**，讓硬體自動完成最後的暫存器還原和模式切換，軟體只需設定好 PSP 並觸發 `bx r14` 即可。

---

# 參考資料

## 分析的原始碼檔案

| 檔案 | 路徑 | 關鍵行號 |
|------|------|----------|
| port.c | `portable/GCC/ARM_CM4F/port.c` | 505-559, 562-586 |
| portmacro.h | `portable/GCC/ARM_CM4F/portmacro.h` | 88-114 |
| tasks.c | `FreeRTOS-Kernel/tasks.c` | 5056-5139 |

## 追蹤的關鍵函式

| 函式 | 用途 |
|------|------|
| `xPortSysTickHandler()` | SysTick 中斷，觸發 PendSV |
| `xPortPendSVHandler()` | Context Switch 核心，儲存/還原暫存器 |
| `vTaskSwitchContext()` | 排程器，選擇下一個任務 |
| `taskSELECT_HIGHEST_PRIORITY_TASK()` | 優先權選擇巨集 |

## 硬體暫存器

| 暫存器 | 用途 |
|--------|------|
| PSP | Process Stack Pointer（任務堆疊） |
| MSP | Main Stack Pointer（例外處理器堆疊） |
| BASEPRI | 中斷優先權遮蔽 |
| NVIC_ICSR | 中斷控制（bit 28 = PENDSVSET） |
| EXC_RETURN | 例外返回控制（位於 LR/R14） |

---

*本文件為 CCU 作業系統期末專題 - Q6 分析*  
*分析基於 FreeRTOS Kernel V11.1.0 (ARM_CM4F port)*  
*日期：2026-01-09*
