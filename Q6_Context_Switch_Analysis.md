# Q6: FreeRTOS Context Switch 實作分析

## 題目

追蹤 FreeRTOS 的 Context Switch 實作，並解釋：
1. 哪些暫存器由硬體儲存/還原，哪些由軟體處理
2. PendSV handler 如何與 vTaskSwitchContext() 互動
3. 新任務的堆疊指標如何在返回 Thread mode 之前被還原

---

## 概述

本文分析 FreeRTOS 在 ARM Cortex-M4F 架構上的 Context Switch 實作機制。Context Switch（上下文切換）是作業系統最核心的功能之一，它允許 CPU 在不同的任務之間切換執行，實現多工處理。

**主要原始碼檔案：**
- `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c`
- `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/portmacro.h`
- `FreeRTOS/FreeRTOS-Kernel/tasks.c`

---

## 1. 硬體與軟體的暫存器儲存/還原分工

### 1.1 硬體自動儲存的暫存器

當 ARM Cortex-M4F 處理器進入例外（如 PendSV）時，**硬體會自動**將以下暫存器推入堆疊：

| 暫存器 | 說明 |
|--------|------|
| **R0-R3** | 引數/臨時暫存器（caller-saved） |
| **R12** | 過程間呼叫臨時暫存器 |
| **LR (R14)** | 連結暫存器（返回位址） |
| **PC (R15)** | 程式計數器 |
| **xPSR** | 程式狀態暫存器 |
| **S0-S15** | FPU 暫存器（若 FPU 啟用，使用 Lazy Stacking） |

這些暫存器會被自動推入 **Process Stack Pointer (PSP)** 所指向的堆疊中。

### 1.2 軟體手動儲存的暫存器

PendSV handler 需要**手動**儲存以下暫存器：

| 暫存器 | 說明 |
|--------|------|
| **R4-R11** | Callee-saved 暫存器（必須跨函式呼叫保留） |
| **R14 (EXC_RETURN)** | 例外返回值（決定返回模式） |
| **S16-S31** | 高位 FPU 暫存器（若有使用 FPU） |

### 1.3 關鍵程式碼片段

**檔案：** `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c`  
**函式：** `xPortPendSVHandler()` (第 505-559 行)

```c
void xPortPendSVHandler( void )
{
    __asm volatile
    (
        // === 取得當前堆疊指標（硬體已自動儲存 R0-R3, R12, LR, PC, xPSR） ===
        "   mrs r0, psp                         \n"  // R0 = 當前任務的 PSP
        "   isb                                 \n"
        
        // === 取得當前 TCB ===
        "   ldr r3, pxCurrentTCBConst           \n"  // R3 = &pxCurrentTCB
        "   ldr r2, [r3]                        \n"  // R2 = pxCurrentTCB
        
        // === 軟體儲存 FPU 高位暫存器（若有使用） ===
        "   tst r14, #0x10                      \n"  // 測試 EXC_RETURN 的 bit 4
        "   it eq                               \n"  // 若 bit 4 = 0，表示使用了 FPU
        "   vstmdbeq r0!, {s16-s31}             \n"  // 儲存 S16-S31
        
        // === 軟體儲存 R4-R11 和 EXC_RETURN ===
        "   stmdb r0!, {r4-r11, r14}            \n"  // 推入堆疊
        "   str r0, [r2]                        \n"  // 更新 TCB->pxTopOfStack
        // ...
    );
}
```

### 1.4 暫存器儲存總結圖

```
╔═══════════════════════════════════════════════════════════════╗
║                    任務堆疊記憶體配置                          ║
╚═══════════════════════════════════════════════════════════════╝

高位址
    ┌─────────────────┐
    │   任務資料       │ ◄── 堆疊基底
    ├─────────────────┤
    │   可用空間       │
    ├─────────────────┤ ◄── PSP（例外發生前）
    │                 │
    │  ╔═════════════╗│
    │  ║  xPSR       ║│
    │  ║  PC         ║│
    │  ║  LR         ║│  ◄── 硬體自動儲存
    │  ║  R12        ║│      （Exception Entry）
    │  ║  R3         ║│
    │  ║  R2         ║│
    │  ║  R1         ║│
    │  ║  R0         ║│
    │  ╚═════════════╝│
    ├─────────────────┤ ◄── PSP（硬體 stacking 後）
    │  (S0-S15)       │     FPU Lazy Stacking（若有使用）
    ├─────────────────┤
    │                 │
    │  ╔═════════════╗│
    │  ║  R11        ║│
    │  ║  R10        ║│
    │  ║  R9         ║│
    │  ║  R8         ║│  ◄── 軟體手動儲存
    │  ║  R7         ║│      （PendSV Handler）
    │  ║  R6         ║│
    │  ║  R5         ║│
    │  ║  R4         ║│
    │  ║  R14(EXC)   ║│
    │  ╚═════════════╝│
    ├─────────────────┤
    │  (S16-S31)      │     軟體儲存（若有使用 FPU）
    ├─────────────────┤ ◄── TCB->pxTopOfStack（最終儲存值）
    │                 │
    ▼                 ▼
低位址
```

---

## 2. PendSV Handler 與 vTaskSwitchContext() 的互動機制

### 2.1 為什麼使用 PendSV？

PendSV（Pendable Service Call）是 ARM Cortex-M 架構中專門設計用於 Context Switch 的例外：

1. **最低優先權**：PendSV 被設定為最低優先權的例外，確保不會中斷其他重要的中斷服務程式
2. **延遲執行**：可以被「掛起」，等到所有高優先權中斷處理完畢後才執行
3. **原子性**：確保 Context Switch 的完整性

### 2.2 觸發流程

**檔案：** `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c`  
**函式：** `xPortSysTickHandler()` (第 562-586 行)

```c
void xPortSysTickHandler( void )
{
    portDISABLE_INTERRUPTS();
    traceISR_ENTER();
    {
        // 遞增系統 tick
        if( xTaskIncrementTick() != pdFALSE )
        {
            traceISR_EXIT_TO_SCHEDULER();
            
            // 需要 Context Switch，觸發 PendSV
            portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT;  // 設定 PENDSVSET 位元
        }
        else
        {
            traceISR_EXIT();
        }
    }
    portENABLE_INTERRUPTS();
}
```

**portYIELD() 巨集定義：**

**檔案：** `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/portmacro.h` (第 88-97 行)

```c
#define portYIELD()                                     \
    {                                                   \
        /* 設定 PendSV 來請求 Context Switch */          \
        portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT; \
        __asm volatile ( "dsb" ::: "memory" );          \
        __asm volatile ( "isb" );                       \
    }

#define portNVIC_INT_CTRL_REG     ( *( ( volatile uint32_t * ) 0xe000ed04 ) )
#define portNVIC_PENDSVSET_BIT    ( 1UL << 28UL )
```

### 2.3 PendSV 與 vTaskSwitchContext() 互動

**三階段架構：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     PendSV Handler（組合語言）                   │
│                     port.c 第 505-559 行                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    ┌──────────────────┐                                         │
│    │  第一階段：儲存   │                                         │
│    │  舊任務的 Context │                                         │
│    └────────┬─────────┘                                         │
│             │                                                    │
│             ▼                                                    │
│    ┌──────────────────┐      ┌─────────────────────────────┐   │
│    │  關閉中斷         │      │  vTaskSwitchContext()       │   │
│    │  (BASEPRI)       │─────▶│  tasks.c 第 5056-5139 行    │   │
│    │                  │      │                             │   │
│    │  呼叫排程器       │      │  • 檢查排程器狀態           │   │
│    │                  │      │  • 更新執行時間統計         │   │
│    │  開啟中斷         │◀─────│  • 檢查堆疊溢位            │   │
│    └────────┬─────────┘      │  • 選擇最高優先權任務       │   │
│             │                 │  • 更新 pxCurrentTCB        │   │
│             ▼                 └─────────────────────────────┘   │
│    ┌──────────────────┐                                         │
│    │  第三階段：還原   │                                         │
│    │  新任務的 Context │                                         │
│    └──────────────────┘                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.4 關鍵程式碼：呼叫排程器

**檔案：** `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c`  
**函式：** `xPortPendSVHandler()` (第 524-532 行)

```c
        // === 呼叫排程器（C 函式） ===
        "   stmdb sp!, {r0, r3}                 \n"  // 保存工作暫存器到 MSP
        "   mov r0, %0                          \n"  // R0 = configMAX_SYSCALL_INTERRUPT_PRIORITY
        "   msr basepri, r0                     \n"  // 遮蔽中斷（進入臨界區）
        "   dsb                                 \n"  // 資料同步屏障
        "   isb                                 \n"  // 指令同步屏障
        "   bl vTaskSwitchContext               \n"  // 呼叫排程器
        "   mov r0, #0                          \n"  
        "   msr basepri, r0                     \n"  // 解除中斷遮蔽（離開臨界區）
        "   ldmia sp!, {r0, r3}                 \n"  // 還原工作暫存器
```

### 2.5 vTaskSwitchContext() 函式

**檔案：** `FreeRTOS/FreeRTOS-Kernel/tasks.c`  
**函式：** `vTaskSwitchContext()` (第 5056-5139 行)

```c
void vTaskSwitchContext( void )
{
    traceENTER_vTaskSwitchContext();

    if( uxSchedulerSuspended != ( UBaseType_t ) 0U )
    {
        // 排程器暫停中，延遲 Context Switch
        xYieldPendings[ 0 ] = pdTRUE;
    }
    else
    {
        xYieldPendings[ 0 ] = pdFALSE;
        traceTASK_SWITCHED_OUT();

        #if ( configGENERATE_RUN_TIME_STATS == 1 )
        {
            // 更新執行時間統計
            ulTotalRunTime[ 0 ] = portGET_RUN_TIME_COUNTER_VALUE();
            if( ulTotalRunTime[ 0 ] > ulTaskSwitchedInTime[ 0 ] )
            {
                pxCurrentTCB->ulRunTimeCounter += 
                    ( ulTotalRunTime[ 0 ] - ulTaskSwitchedInTime[ 0 ] );
            }
            ulTaskSwitchedInTime[ 0 ] = ulTotalRunTime[ 0 ];
        }
        #endif

        // 檢查堆疊溢位
        taskCHECK_FOR_STACK_OVERFLOW();

        // 儲存 errno（若啟用 POSIX）
        #if ( configUSE_POSIX_ERRNO == 1 )
        {
            pxCurrentTCB->iTaskErrno = FreeRTOS_errno;
        }
        #endif

        // ★★★ 核心操作：選擇最高優先權的任務 ★★★
        // 這會更新 pxCurrentTCB 指向新任務的 TCB
        taskSELECT_HIGHEST_PRIORITY_TASK();
        
        traceTASK_SWITCHED_IN();
        portTASK_SWITCH_HOOK( pxCurrentTCB );

        // 還原新任務的 errno
        #if ( configUSE_POSIX_ERRNO == 1 )
        {
            FreeRTOS_errno = pxCurrentTCB->iTaskErrno;
        }
        #endif
    }

    traceRETURN_vTaskSwitchContext();
}
```

### 2.6 為什麼要分開 Assembly 和 C？

| 層級 | 語言 | 負責工作 | 原因 |
|------|------|----------|------|
| PendSV Handler | 組合語言 | 暫存器操作、堆疊管理 | 需要直接存取 CPU 暫存器 |
| vTaskSwitchContext | C 語言 | 排程邏輯、任務選擇 | 可移植性高、易於維護 |

**優點：**
1. **可移植性**：排程邏輯（C）可跨不同 ARM Cortex-M 變體使用
2. **可維護性**：修改排程演算法不需要更動組合語言
3. **安全性**：BASEPRI 臨界區保護 pxCurrentTCB 的更新

---

## 3. 新任務堆疊指標的還原流程

### 3.1 三階段還原過程

#### 階段一：載入新任務的堆疊指標

**程式碼位置：** `port.c` 第 534-535 行

```assembly
ldr r1, [r3]      ; r3 = &pxCurrentTCB（全域變數位址）
                  ; r1 = pxCurrentTCB（新任務的 TCB 位址）
ldr r0, [r1]      ; r0 = TCB->pxTopOfStack（新任務的堆疊指標）
```

**說明：**
- `r3` 包含 `pxCurrentTCB` 的位址（在呼叫 vTaskSwitchContext 前已載入）
- `r1` 載入新任務的 TCB 位址（已被 vTaskSwitchContext 更新）
- `r0` 載入 TCB 的第一個欄位 `pxTopOfStack`

#### 階段二：還原軟體儲存的暫存器

**程式碼位置：** `port.c` 第 537-541 行

```assembly
ldmia r0!, {r4-r11, r14}    ; 從堆疊彈出 R4-R11 和 EXC_RETURN
                            ; r0 自動遞增

tst r14, #0x10              ; 測試 EXC_RETURN 的 bit 4
it eq                       ; 若 bit 4 = 0（使用了 FPU）
vldmiaeq r0!, {s16-s31}     ; 還原高位 FPU 暫存器
                            ; r0 自動遞增
```

**說明：**
- `ldmia r0!, {r4-r11, r14}` 彈出 9 個字組（36 bytes）
- R0 自動遞增指向下一個區塊
- 條件式還原 FPU 暫存器（若 bit 4 = 0）

#### 階段三：設定 PSP 並觸發硬體還原

**程式碼位置：** `port.c` 第 543-553 行

```assembly
msr psp, r0       ; 設定 Process Stack Pointer
isb               ; 指令同步屏障（確保 PSP 更新完成）

bx r14            ; 跳轉到 EXC_RETURN 值
                  ; 觸發例外返回
```

### 3.2 EXC_RETURN 值的意義

當 `bx r14` 執行時，處理器會檢測到 EXC_RETURN 的特殊值並執行例外返回：

**FreeRTOS 使用的 EXC_RETURN 值：0xFFFFFFFD**

| 位元 | 值 | 意義 |
|------|-----|------|
| [31:4] | 0xFFFFFFF | 例外返回標記 |
| [3] | 1 | 返回 Thread mode（非 Handler mode） |
| [2] | 1 | 使用 PSP（非 MSP） |
| [4] | 1 | 標準框架（或 Lazy FPU stacking） |

**硬體自動執行的操作：**
1. 從 PSP 彈出 R0, R1, R2, R3, R12, LR, PC, xPSR（8 個字組 = 32 bytes）
2. 若有 FPU 框架，彈出 S0-S15
3. 將 PC 載入彈出的返回位址 → 任務恢復執行
4. 切換至 Thread mode
5. 切換使用 PSP（而非 MSP）

### 3.3 堆疊指標還原示意圖

```
                        TCB (新任務)
pxCurrentTCB ─────────▶ ┌─────────────────┐
                        │ pxTopOfStack ───┼──────┐
                        ├─────────────────┤      │
                        │ xStateListItem  │      │
                        │ ...             │      │
                        └─────────────────┘      │
                                                  │
                        新任務的堆疊記憶體         │
                        ┌─────────────────┐      │
                        │ R4              │◀─────┘ ① ldr r0, [r1]
                        │ R5              │         r0 指向這裡
                        │ R6              │
                        │ R7              │
                        │ R8              │
                        │ R9              │
                        │ R10             │
                        │ R11             │
                        │ R14 (EXC_RET)   │
                        ├─────────────────┤◀───── ② ldmia r0!, {...}
                        │ (S16-S31)       │         彈出後 r0 指向這裡
                        ├─────────────────┤◀───── ③ vldmiaeq r0!, {...}
                        │ R0              │         彈出後 r0 指向這裡
                        │ R1              │
                        │ R2              │
                        │ R3              │
                        │ R12             │
                        │ LR              │
                        │ PC              │ ◀── 任務恢復執行的位址
                        │ xPSR            │
                        ├─────────────────┤◀───── ④ msr psp, r0
                        │ (S0-S15)        │         PSP 設定到這裡
                        └─────────────────┘
                                                  ⑤ bx r14
                                                  硬體彈出 R0-R3, R12, LR, PC, xPSR
                                                  PC 被載入，任務恢復執行
```

---

## 4. 完整的 Context Switch 流程追蹤

### 4.1 函式呼叫序列

```
1. SysTick Timer 中斷觸發
   └─▶ xPortSysTickHandler()                    [port.c:562-586]
       └─▶ xTaskIncrementTick()                 [tasks.c]
           └─▶ 返回 pdTRUE（需要 Context Switch）
               └─▶ portNVIC_INT_CTRL_REG = portNVIC_PENDSVSET_BIT
                   （觸發 PendSV 例外）

2. PendSV 例外觸發（最低優先權）
   ├─▶ [硬體自動儲存 R0-R3, R12, LR, PC, xPSR 到 PSP]
   │
   └─▶ xPortPendSVHandler()                     [port.c:505-559]
       │
       ├─▶ [儲存舊任務 Context]
       │   ├─▶ mrs r0, psp                      取得當前 PSP
       │   ├─▶ ldr r2, [pxCurrentTCB]           取得舊任務 TCB
       │   ├─▶ vstmdbeq r0!, {s16-s31}          儲存 FPU（若有）
       │   ├─▶ stmdb r0!, {r4-r11, r14}         儲存 R4-R11, EXC_RETURN
       │   └─▶ str r0, [r2]                     更新 TCB->pxTopOfStack
       │
       ├─▶ [呼叫排程器]
       │   ├─▶ msr basepri, MAX_PRI             關閉中斷
       │   ├─▶ bl vTaskSwitchContext ─────────────────────┐
       │   │                                               │
       │   │   vTaskSwitchContext()             [tasks.c:5056-5139]
       │   │   ├─▶ 檢查排程器狀態                          │
       │   │   ├─▶ 更新執行時間統計                        │
       │   │   ├─▶ 檢查堆疊溢位                            │
       │   │   ├─▶ taskSELECT_HIGHEST_PRIORITY_TASK()     │
       │   │   │   └─▶ 更新 pxCurrentTCB 為新任務          │
       │   │   └─▶ 返回 ◀──────────────────────────────────┘
       │   │
       │   └─▶ msr basepri, #0                  開啟中斷
       │
       ├─▶ [還原新任務 Context]
       │   ├─▶ ldr r1, [pxCurrentTCB]           取得新任務 TCB
       │   ├─▶ ldr r0, [r1]                     取得新任務 pxTopOfStack
       │   ├─▶ ldmia r0!, {r4-r11, r14}         還原 R4-R11, EXC_RETURN
       │   ├─▶ vldmiaeq r0!, {s16-s31}          還原 FPU（若有）
       │   ├─▶ msr psp, r0                      設定 PSP
       │   └─▶ bx r14                           例外返回
       │
       └─▶ [硬體自動還原 R0-R3, R12, LR, PC, xPSR 從 PSP]

3. 新任務恢復執行
   └─▶ PC 載入新任務的返回位址，繼續執行
```

### 4.2 時序圖

```
時間 ──────────────────────────────────────────────────────────────────▶

任務 A     │ SysTick    │        PendSV Handler         │    任務 B
執行中     │ 中斷       │                               │    執行中
           │            │                               │
   ████████│            │                               │████████
           │            │                               │
           │◀──觸發────▶│◀───────────────────────────▶│
           │            │                               │
           │  HW        │  SW      排程器    SW      HW │
           │  儲存      │  儲存              還原    還原│
           │            │                               │
           │ R0-R3      │ R4-R11  vTaskSwitch  R4-R11  R0-R3
           │ R12,LR     │ R14     Context()    R14     R12,LR
           │ PC,xPSR    │ S16-31              S16-31   PC,xPSR
           │            │                               │
           │            │         pxCurrentTCB          │
           │            │         TCB_A → TCB_B         │
           │            │                               │
```

---

## 5. 設計原理說明

### 5.1 為什麼使用 PendSV 而不是直接在 SysTick 中切換？

1. **優先權考量**：SysTick 可能被設為較高優先權，直接在其中執行 Context Switch 會延遲其他高優先權中斷
2. **延遲執行**：PendSV 設為最低優先權，確保所有其他中斷都處理完畢後才執行
3. **原子性保證**：Context Switch 過程不會被打斷

### 5.2 為什麼需要 BASEPRI 臨界區？

```c
msr basepri, configMAX_SYSCALL_INTERRUPT_PRIORITY  // 遮蔽中斷
bl vTaskSwitchContext()                            // 安全區域
msr basepri, #0                                    // 解除遮蔽
```

**原因：**
- 確保 `pxCurrentTCB` 的更新是原子操作
- 防止其他中斷在排程過程中修改全域狀態
- 保證排程決策的一致性

### 5.3 Lazy FPU Stacking 的優化

FreeRTOS 使用 ARM Cortex-M 的 Lazy FPU Stacking 功能：

**啟用設定：** `port.c` 第 449 行
```c
*( portFPCCR ) |= portASPEN_AND_LSPEN_BITS;
```

**優點：**
- 只有在任務實際使用 FPU 時才儲存 FPU 暫存器
- 對於不使用 FPU 的任務，減少 Context Switch 開銷
- 硬體自動偵測並處理

---

## 6. 總結

FreeRTOS 在 ARM Cortex-M4F 上的 Context Switch 實作展現了**硬體-軟體協作**的精妙設計：

### 核心要點

1. **硬體與軟體分工**
   - 硬體自動處理 volatile 暫存器（R0-R3, R12, LR, PC, xPSR）的儲存與還原
   - 軟體負責 non-volatile 暫存器（R4-R11, R14, S16-S31）的處理
   - 這種分工最大化效率，同時確保完整性

2. **PendSV 與 vTaskSwitchContext 的互動**
   - PendSV Handler（組合語言）處理低階暫存器/堆疊操作
   - vTaskSwitchContext()（C 語言）處理高階排程邏輯
   - BASEPRI 臨界區保護 pxCurrentTCB 的原子更新
   - 這種分離提供可移植性和可維護性

3. **堆疊指標還原機制**
   - 三階段過程：載入 TCB、彈出軟體儲存的暫存器、設定 PSP 並觸發例外返回
   - EXC_RETURN 魔術值（0xFFFFFFFD）指示硬體返回 Thread mode 並使用 PSP
   - 硬體自動完成最後的暫存器還原和模式切換

### 設計優勢

- **高效能**：利用硬體自動 stacking，最小化軟體開銷
- **可移植性**：排程邏輯（C）可在不同 ARM Cortex-M 變體間移植
- **即時性**：PendSV 最低優先權確保不影響關鍵中斷
- **FPU 優化**：Lazy Stacking 減少非 FPU 任務的切換成本

---

## 參考資料

### 分析的原始碼檔案
1. **Port 層（ARM Cortex-M4F）**：
   - `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/port.c` (第 505-586 行)
   - `FreeRTOS/FreeRTOS-Kernel/portable/GCC/ARM_CM4F/portmacro.h` (第 88-114 行)

2. **核心層**：
   - `FreeRTOS/FreeRTOS-Kernel/tasks.c` (第 5056-5139 行: `vTaskSwitchContext`)

### 追蹤的關鍵函式
- `xPortSysTickHandler()` - 觸發 PendSV
- `xPortPendSVHandler()` - 執行實際的 Context Switch
- `vTaskSwitchContext()` - 選擇下一個任務
- `taskSELECT_HIGHEST_PRIORITY_TASK()` - 優先權選擇

### 使用的硬體暫存器
- **PSP**：Process Stack Pointer（任務 Context）
- **MSP**：Main Stack Pointer（例外處理器）
- **BASEPRI**：排程器呼叫時的中斷遮蔽
- **NVIC**：中斷控制（PendSV 觸發）
- **EXC_RETURN**：例外返回行為控制

---

*本文件為 CCU 作業系統期末專題 - Q6 分析*  
*分析基於 FreeRTOS Kernel V11.1.0 (ARM_CM4F port)*  
*日期：2026-01-09*
