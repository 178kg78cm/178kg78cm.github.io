---
layout: post
title:  "임베디드MCU : Timer Interrupt.md"
date:   2025-02-13 +09001
categories: 임베디드MCU
tag: "Infineon"
---



## Timer Interrupt

### 타이머 초기화 코드 분석


- Timer 초기화 코드 분석

1. Config 초기화
2. COnfig 설정
3. Config 적용 / Register 변경
   
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