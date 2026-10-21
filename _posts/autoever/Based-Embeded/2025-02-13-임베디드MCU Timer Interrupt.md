---
layout: post
title: "임베디드 MCU : Timer Interrupt와 FSM, Task Scheduler"
description: "TC275 타이머 인터럽트 초기화 코드를 분석하고, FSM으로 신호등을 만든 뒤 간단한 Task Scheduler를 구현합니다."
date:   2025-02-13 +0900
categories: Embedded
tags: [TC275]
---



## Timer Interrupt

`delay()`처럼 CPU가 숫자를 세며 기다리는 방식은 그동안 아무것도 할 수 없고, 최적화나 클럭에 따라 시간이 바뀐다. **하드웨어 타이머가 시간을 재고, 때가 되면 인터럽트로 알려 주는** 방식이 정확하고 효율적이다.

TC275의 **STM(System Timer)** 은 계속 증가하는 64비트 카운터다. 비교 레지스터(Compare)에 값을 넣어 두면, 카운터가 그 값에 도달할 때 인터럽트가 발생한다.

### 타이머 초기화 코드 분석

- Timer 초기화 코드 분석 (iLLD의 공통 패턴)

1. **Config 초기화** : `IfxStm_initCompareConfig()`로 설정 구조체를 기본값으로 채운다.
2. **Config 설정** : 필요한 항목만 바꾼다. (인터럽트 우선순위, 처리할 CPU, 몇 tick 뒤에 발생할지)
3. **Config 적용 / Register 변경** : `IfxStm_initCompare()`가 구조체 내용을 실제 레지스터에 쓴다.

- `ticks = 100000000` : STM 클럭이 100MHz라면 1초다.
- ISR 안에서 `IfxStm_increaseCompare()`로 **다음 비교 값을 현재 값 + 주기**로 다시 설정해야 주기적으로 인터럽트가 발생한다. 플래그를 지우지 않으면 ISR이 계속 다시 호출된다.
- 아래 코드는 1초마다 P10.2 LED를 토글한다.
   
``` c 
#include "Ifx_Types.h"
#include "IfxCpu.h"
#include "IfxScuWdt.h"

#include "IfxPort.h"
#include "IfxPort_PinMap.h"

#include "IfxStm.h"
#include "IfxCpu_Irq.h"

typedef struct
{
    Ifx_STM             *stmSfr;            /**< \brief Pointer to Stm register base */
    IfxStm_CompareConfig stmConfig;         /**< \brief Stm Configuration structure */
    volatile uint8       LedBlink;          /**< \brief LED state variable */
    volatile uint32      counter;           /**< \brief interrupt counter */
} App_Stm;

App_Stm g_Stm; /**< \brief Stm global data */

IfxCpu_syncEvent g_cpuSyncEvent = 0;

void IfxStmDemo_init(void);

int core0_main(void)
{
    IfxCpu_enableInterrupts();
    
    /* !!WATCHDOG0 AND SAFETY WATCHDOG ARE DISABLED HERE!!
     * Enable the watchdogs and service them periodically if it is required
     */
    IfxScuWdt_disableCpuWatchdog(IfxScuWdt_getCpuWatchdogPassword());
    IfxScuWdt_disableSafetyWatchdog(IfxScuWdt_getSafetyWatchdogPassword());
    
    /* Wait for CPU sync event */
    IfxCpu_emitEvent(&g_cpuSyncEvent);
    IfxCpu_waitEvent(&g_cpuSyncEvent, 1);

    IfxStmDemo_init();

    /*P00_5    Digital Output*/
    IfxPort_setPinModeOutput(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex, IfxPort_OutputMode_pushPull, IfxPort_OutputIdx_general);
    IfxPort_setPinLow(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);

    while(1)
    {

    }
    return (1);
}

IFX_INTERRUPT(STM_Int0Handler, 0, 100);

void STM_Int0Handler(void)
{
    static int flag = 0;
    static int cnt = 0;

    IfxStm_clearCompareFlag(g_Stm.stmSfr, g_Stm.stmConfig.comparator);
    IfxStm_increaseCompare(g_Stm.stmSfr, g_Stm.stmConfig.comparator, 100000000u);

    cnt++;

    if(flag == 0)
    {
        IfxPort_setPinLow(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
        flag = 1;
    }
    else
    {
        IfxPort_setPinHigh(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
        flag = 0;
    }

    IfxCpu_enableInterrupts();
}

void IfxStmDemo_init(void)
{
    /* disable interrupts */
    boolean interruptState = IfxCpu_disableInterrupts();

    IfxStm_enableOcdsSuspend(&MODULE_STM0);

    g_Stm.stmSfr = &MODULE_STM0;
    IfxStm_initCompareConfig(&g_Stm.stmConfig);

    g_Stm.stmConfig.triggerPriority = 100u;
    g_Stm.stmConfig.typeOfService   = IfxSrc_Tos_cpu0;
    g_Stm.stmConfig.ticks           = 100000000;

    IfxStm_initCompare(g_Stm.stmSfr, &g_Stm.stmConfig);

    /* enable interrupts again */
    IfxCpu_restoreInterrupts(interruptState);
}

```
### FSM

FSM(Finite State Machine, 유한 상태 기계)은 시스템을 **정해진 상태들과, 이벤트에 따른 상태 전이**로 설계하는 방법이다.

```text
            버튼 (ISR0)
  ┌──────┐ ────────────▶ ┌───────┐
  │ STOP │               │ BLINK │
  └──────┘ ◀──────────── └───────┘
            버튼 (ISR0)
```

- **이벤트 처리(ISR)는 상태만 바꾸고**, 실제 동작(LED 제어)은 상태를 보고 타이머 ISR이나 메인 루프에서 한다.
- 이렇게 나누면 ISR이 짧아지고, 상태가 늘어나도 `switch(state)`에 case만 추가하면 된다.
- 아래 신호등 실습이 이 구조다. 버튼 인터럽트(ISR0)는 STOP ↔ BLINK만 바꾸고, 타이머 인터럽트가 상태에 따라 LED 패턴을 진행한다.

## 신호등 만들기
- Interrupt 하나로 만들기 ( Cmp 1개 사용 )
``` c
// Interrupt 하나로 만들기 ( Cmp 1개 사용 )
#include "Ifx_Types.h"
#include "IfxCpu.h"
#include "IfxScuWdt.h"

#include "IfxPort.h"
#include "IfxPort_PinMap.h"

#include "IfxStm.h"
#include "IfxCpu_Irq.h"
// ERU related
#define EXIS0_IDX 4
#define FEN0_IDX 8
#define EIEN0_IDX 11
#define INP0_IDX  12
#define IGP0_IDX  14

// SRC related
#define SRE_IDX 10
#define TOS_IDX 11

#define STOP 1
#define BLINK 0
int state = 0;
void initERU(void);
typedef struct
{
    Ifx_STM             *stmSfr;            /**< \brief Pointer to Stm register base */
    IfxStm_CompareConfig stmConfig;         /**< \brief Stm Configuration structure */
    volatile uint8       LedBlink;          /**< \brief LED state variable */
    volatile uint32      counter;           /**< \brief interrupt counter */
} App_Stm;

App_Stm g_Stm; /**< \brief Stm global data */

IfxCpu_syncEvent g_cpuSyncEvent = 0;

void IfxStmDemo_init(void);

int core0_main(void)
{
    IfxCpu_enableInterrupts();
    
    /* !!WATCHDOG0 AND SAFETY WATCHDOG ARE DISABLED HERE!!
     * Enable the watchdogs and service them periodically if it is required
     */
    IfxScuWdt_disableCpuWatchdog(IfxScuWdt_getCpuWatchdogPassword());
    IfxScuWdt_disableSafetyWatchdog(IfxScuWdt_getSafetyWatchdogPassword());

    /* Wait for CPU sync event */
    IfxCpu_emitEvent(&g_cpuSyncEvent);
    IfxCpu_waitEvent(&g_cpuSyncEvent, 1);

    IfxStmDemo_init();

    /*P00_5    Digital Output*/
    IfxPort_setPinModeOutput(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex, IfxPort_OutputMode_pushPull, IfxPort_OutputIdx_general);
    IfxPort_setPinModeOutput(IfxPort_P10_1.port, IfxPort_P10_1.pinIndex, IfxPort_OutputMode_pushPull, IfxPort_OutputIdx_general);
    
    IfxPort_setPinLow(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
    IfxPort_setPinLow(IfxPort_P10_1.port, IfxPort_P10_1.pinIndex);

    initERU();
    while(1)
    {

    }
    return (1);
}
IFX_INTERRUPT(ISR0,0,0x10);
IFX_INTERRUPT(STM_Int0Handler, 0, 100);
void ISR0(void){
    //state transition
    switch(state){
        case STOP:
            state = BLINK;
            break;
        case BLINK:
            state = STOP;
            break;
        default:
            state = STOP;
    }
}
void STM_Int0Handler(void)
{
    static int flag = 0;
    static int cnt = 0;

    // Compare Flag를 초기화하고
    IfxStm_clearCompareFlag(g_Stm.stmSfr, g_Stm.stmConfig.comparator);
    // Compare 관련 register 즉 0를 다시 셋팅
    IfxStm_increaseCompare(g_Stm.stmSfr, g_Stm.stmConfig.comparator, 50000000u);
        /// 2번하고 4번만 인터럽트 동작
    cnt++;
    if(state == STOP) cnt --;

    if(cnt==10){
        if(flag == 0)
        {
            IfxPort_setPinLow(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
            IfxPort_setPinHigh(IfxPort_P10_1.port, IfxPort_P10_1.pinIndex);
            flag = 1;
        }
        else if(flag == 1)
        {
            IfxPort_setPinHigh(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
            IfxPort_setPinLow(IfxPort_P10_1.port, IfxPort_P10_1.pinIndex);
            flag = 2;
        }else if(flag == 2){
            IfxPort_togglePin(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
            flag = 0;
        }
    }
    IfxCpu_enableInterrupts();
}


void IfxStmDemo_init(void)
{
    /* disable interrupts */
    boolean interruptState = IfxCpu_disableInterrupts();

    IfxStm_enableOcdsSuspend(&MODULE_STM0);

    g_Stm.stmSfr = &MODULE_STM0; // STM에 0번 모듈 사용하겠다.
    IfxStm_initCompareConfig(&g_Stm.stmConfig); // SstmConfig에 필요한 값 세팅

    g_Stm.stmConfig.triggerPriority = 100u; // Interrupt Priority 0~0xff
    g_Stm.stmConfig.typeOfService   = IfxSrc_Tos_cpu0;
    g_Stm.stmConfig.ticks           = 50000000; // 0.5초 // STM기본 주파수 100Mhz //
    IfxStm_initCompare(g_Stm.stmSfr, &g_Stm.stmConfig);

    /* enable interrupts again */
    IfxCpu_restoreInterrupts(interruptState);
}


void initERU(void){
    SCU_EICR1.U &= ~(0x7 << EXIS0_IDX);
    SCU_EICR1.U |= 0x1 << EXIS0_IDX;
    SCU_EICR1.U |= 0x1 << FEN0_IDX;
    SCU_EICR1.U |= 0x1 << EIEN0_IDX;
    SCU_EICR1.U &= ~(0x7 << INP0_IDX);

    SCU_IGCR0.U &= ~(0x3 << IGP0_IDX);
    SCU_IGCR0.U |= 0x1 << IGP0_IDX;

    SRC_SCU_SCU_ERU0.U &= ~0XFF;
    SRC_SCU_SCU_ERU0.U |= 0X10;
    SRC_SCU_SCU_ERU0.U |= 0x1 << SRE_IDX;
    SRC_SCU_SCU_ERU0.U &= ~(0x3 << TOS_IDX);
}


```

- Interrupt 두개로 만들기 ( Cmp 2개 사용 )
```c
// Interrupt 두개로 만들기 ( Cmp 2개 사용 )
#include "Ifx_Types.h"
#include "IfxCpu.h"
#include "IfxScuWdt.h"

#include "IfxPort.h"
#include "IfxPort_PinMap.h"

#include "IfxStm.h"
#include "IfxCpu_Irq.h"
// ERU related
#define EXIS0_IDX 4
#define FEN0_IDX 8
#define EIEN0_IDX 11
#define INP0_IDX  12
#define IGP0_IDX  14

// SRC related
#define SRE_IDX 10
#define TOS_IDX 11

#define STOP 0
#define ACTION 1

static int state = 1;
void initERU(void);
typedef struct
{
    Ifx_STM             *stmSfr;            /**< \brief Pointer to Stm register base */
    IfxStm_CompareConfig stmConfig;         /**< \brief Stm Configuration structure */
    volatile uint8       LedBlink;          /**< \brief LED state variable */
    volatile uint32      counter;           /**< \brief interrupt counter */
} App_Stm;

App_Stm g_Stm; /**< \brief Stm global data */
App_Stm t_Stm;
IfxCpu_syncEvent g_cpuSyncEvent = 0;

void IfxStmDemo_init(void);

int core0_main(void)
{
    IfxCpu_enableInterrupts();
    
    /* !!WATCHDOG0 AND SAFETY WATCHDOG ARE DISABLED HERE!!
     * Enable the watchdogs and service them periodically if it is required
     */
    IfxScuWdt_disableCpuWatchdog(IfxScuWdt_getCpuWatchdogPassword());
    IfxScuWdt_disableSafetyWatchdog(IfxScuWdt_getSafetyWatchdogPassword());

    /* Wait for CPU sync event */
    IfxCpu_emitEvent(&g_cpuSyncEvent);
    IfxCpu_waitEvent(&g_cpuSyncEvent, 1);

    IfxStmDemo_init();

    /*P00_5    Digital Output*/
    IfxPort_setPinModeOutput(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex, IfxPort_OutputMode_pushPull, IfxPort_OutputIdx_general);
    IfxPort_setPinModeOutput(IfxPort_P10_1.port, IfxPort_P10_1.pinIndex, IfxPort_OutputMode_pushPull, IfxPort_OutputIdx_general);

    IfxPort_setPinLow(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
    IfxPort_setPinLow(IfxPort_P10_1.port, IfxPort_P10_1.pinIndex);

    initERU();
    while(1)
    {

    }
    return (1);
}
IFX_INTERRUPT(ISR0,0,0x10);
IFX_INTERRUPT(STM_Int0Handler, 0, 100);
IFX_INTERRUPT(STM_Int1Handler, 0, 50);
void ISR0(void){
    //state transition
    switch(state){
        case STOP:
            state = ACTION;
            break;
        case ACTION:
            state = STOP;
            break;
        default:
            state = STOP;
    }
}
static int flag = 0;
void STM_Int0Handler(void)
{
    IfxCpu_enableInterrupts();
    // Compare Flag를 초기화하고
    IfxStm_clearCompareFlag(g_Stm.stmSfr, IfxStm_Comparator_0);
    // Compare 관련 register 즉 0를 다시 셋팅
    IfxStm_increaseCompare(g_Stm.stmSfr, IfxStm_Comparator_0, 500000000u);
        /// 2번하고 4번만 인터럽트 동작
    if(state){
        if(flag == 0)
        {
            IfxPort_setPinLow(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
            IfxPort_setPinHigh(IfxPort_P10_1.port, IfxPort_P10_1.pinIndex);
            flag = 1;
        }
        else if(flag == 1)
        {
            IfxPort_setPinHigh(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
            IfxPort_setPinLow(IfxPort_P10_1.port, IfxPort_P10_1.pinIndex);
            flag = 2;
        }else if(flag == 2){
            flag = 0;
        }
    }
//    IfxPort_togglePin(IfxPort_P10_1.port, IfxPort_P10_1.pinIndex);



}


void STM_Int1Handler(void)
{
    IfxCpu_enableInterrupts();
    // Compare Flag를 초기화하고
    IfxStm_clearCompareFlag(t_Stm.stmSfr, IfxStm_Comparator_1);
    IfxStm_increaseCompare(t_Stm.stmSfr, IfxStm_Comparator_1, 50000000u);

    if(flag==0)
        IfxPort_togglePin(IfxPort_P10_2.port, IfxPort_P10_2.pinIndex);
}


void IfxStmDemo_init(void)
{
    /* disable interrupts */
    boolean interruptState = IfxCpu_disableInterrupts();

    IfxStm_enableOcdsSuspend(&MODULE_STM0);

    // compare 1
    t_Stm.stmSfr = &MODULE_STM0;
    IfxStm_initCompareConfig(&t_Stm.stmConfig); // stmConfig에 필요한 값 세팅

    t_Stm.stmConfig.comparator = IfxStm_Comparator_1;
    t_Stm.stmConfig.triggerPriority = 50u; // Interrupt Priority 0~0xff
    t_Stm.stmConfig.typeOfService   = IfxSrc_Tos_cpu0;
    t_Stm.stmConfig.ticks           = 50000000; // 0.5초 // STM기본 주파수 100Mhz //
    IfxStm_initCompare(t_Stm.stmSfr, &t_Stm.stmConfig);

    // compare 2
    g_Stm.stmSfr = &MODULE_STM0; // STM에 0번 모듈 사용하겠다.
    IfxStm_initCompareConfig(&g_Stm.stmConfig); // stmConfig에 필요한 값 세팅

    g_Stm.stmConfig.triggerPriority = 100u; // Interrupt Priority 0~0xff
    g_Stm.stmConfig.typeOfService   = IfxSrc_Tos_cpu0;
    g_Stm.stmConfig.comparatorInterrupt = IfxStm_ComparatorInterrupt_ir1;
    g_Stm.stmConfig.ticks           = 500000000; // 5초 // STM기본 주파수 100Mhz //
    IfxStm_initCompare(g_Stm.stmSfr, &g_Stm.stmConfig);



    /* enable interrupts again */
    IfxCpu_restoreInterrupts(interruptState);
}


void initERU(void){
    SCU_EICR1.U &= ~(0x7 << EXIS0_IDX);
    SCU_EICR1.U |= 0x1 << EXIS0_IDX;
    SCU_EICR1.U |= 0x1 << FEN0_IDX;
    SCU_EICR1.U |= 0x1 << EIEN0_IDX;
    SCU_EICR1.U &= ~(0x7 << INP0_IDX);

    SCU_IGCR0.U &= ~(0x3 << IGP0_IDX);
    SCU_IGCR0.U |= 0x1 << IGP0_IDX;

    SRC_SCU_SCU_ERU0.U &= ~0XFF;
    SRC_SCU_SCU_ERU0.U |= 0X10;
    SRC_SCU_SCU_ERU0.U |= 0x1 << SRE_IDX;
    SRC_SCU_SCU_ERU0.U &= ~(0x3 << TOS_IDX);
}


```

### Task Scheduler

ISR 하나에 기능을 계속 추가하면 금방 복잡해진다. 그래서 **1ms 타이머 인터럽트 하나로 시간 기준(tick)만 만들고, 그 tick으로 주기별 태스크를 호출하는** 간단한 스케줄러를 만들었다. (RTOS 없이 동작하는 협력형 스케줄러, Time-triggered 구조)

```c
typedef struct
{
    uint32_t u32nuCnt1ms;
    uint32_t u32nuCnt10ms;
    uint32_t u32nuCnt100ms;
    uint32_t u32nuCnt1000ms;
} TestCnt;

TestCnt stTestCnt;
volatile uint32_t g_tick1ms = 0;      /* 1ms STM 인터럽트에서 증가 */

void AppScheduling(void)
{
    static uint32_t lastTick = 0;
    if (g_tick1ms == lastTick) return; /* 새 tick이 없으면 반환 */
    lastTick = g_tick1ms;

    AppTask1ms();                                  /* 1ms마다 */
    if (lastTick % 10   == 0) AppTask10ms();       /* 10ms마다 */
    if (lastTick % 100  == 0) AppTask100ms();      /* 100ms마다 */
    if (lastTick % 1000 == 0) AppTask1000ms();     /* 1s마다 */
}

int core0_main(void)
{
    /* ... 초기화 ... */
    while (1)
    {
        AppScheduling();   /* 메인 루프는 스케줄러만 돌린다 */
    }
}
```

- ISR에서는 tick만 증가시키고, 실제 일은 메인 루프의 태스크에서 한다. ISR을 짧게 유지하는 원칙과 같다.
- 각 태스크의 실행 시간이 다음 tick 전에 끝나야 한다. 한 태스크가 오래 걸리면 다른 태스크가 밀린다. 이 문제를 **우선순위와 선점**으로 해결한 것이 RTOS다. ([RTOS 실습](/2025-01-24-RTOS1/))
- 이 구조는 [SPI 7-Segment 스톱워치](/2025-02-14-임베디드MCU/) 실습에서 그대로 사용했다.
