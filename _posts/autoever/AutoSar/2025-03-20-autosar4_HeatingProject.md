---
layout: post
title:  "Autosar (4) : Heating Project"
date:   2025-03-19 +0900
categories: AUTOSAR
tags: [AUTOSAR]
project:
  name: "AUTOSAR 열선 시트 제어기"
  summary: "SWC 4개(SeatSwitch, SHControl, SeatHeating, HeatingDial)로 열선 시트 제어기를 설계. CAN 명령 수신, 포텐셔미터(ADC) 다이얼, PWM 출력을 Sender-Receiver / Client-Server 포트와 10ms Runnable로 구성하고 IoHwAb로 하드웨어를 분리."
  tech: [AUTOSAR, SWC, RTE, ADC, PWM]
  order: 1
---

## 프로젝트 개요
　신호에 따라서 온열기구를 동작시킨다. 단, Heating기능이 따로 없으므로 LED를 제어하는 방식으로 대체한다. 

- LED1 (Port E Pin 4): ECU가 동작 중임을 표시 (GPIO Out)
- LED2 (Port E Pin 5): 열선 시트가 동작 중임을 표시 (GPIO Out)
- LED4 (Port E Pin 7): 열선 시트 강도 (PWM)
- Potentiometer (Port B Pin 4): 열선 시트 강도 조절용 다이얼 (ADC)

![alt text](../../../assets/images/2025-03-20-autosar4_HeatingProject/practice_ASW_architecture.png)
각 4개의 component를 구현한다. 

- Symbol 설명
  - ▶--▶
    Send - Receiver communication (queue를 사용한다.)
  - O--C
    Server - Client
  - Server runable
    서버에 연결되어 있는 블록으로 client에서 server를 호출할 때마다 동작
  - Runable
    client 사이에 연결되어 있는 블록으로 특정 시간 마다 동작한다. 여기서는 10ms인것을 확인할 수 있다.
  - P / R
    P(Provide) 포트는 데이터나 서비스를 **제공**하고, R(Require) 포트는 **요구**한다.

## 컴포넌트 구성

그림은 크게 두 영역으로 나뉜다. 왼쪽 초록 영역(**CAN HOT**)은 CAN 통신과 연결되고, 오른쪽 빨간 영역(**IoHwAb HOT**)은 IO Hardware Abstraction을 거쳐 실제 핀과 연결된다.

| SWC | Runnable | 주요 포트 | 연결 |
| --- | --- | --- | --- |
| **SeatSwitch** | Re_SeatSwitch | R : RcCommand / P : RcStatus, SeatSwitch | CAN 메시지 `ECU2_Msg_PD` 수신, `ECU1_Msg_SH` 송신 (FlexCAN_B) |
| **SHControl** | Re_SHSwitch, Re_SHControl | R : SeatSwitch / Client : HeatingElement, Setting / P : DialLed | 제어 로직 (다른 SWC만 사용, 하드웨어와 직접 연결 없음) |
| **SeatHeating** | Re_SeatHeating (Server) | Server : HeatingOn / IoHeating | Led4, PE7 (**PWM**) |
| **HeatingDial** | Re_HeatingDial (10ms) | Server : Position / R : DialLed / IoDial, IoLed, IoAliveLed | Potentiometer PB4 (**ADC**), Led1 PE4, Led2 PE5 (**DIO**) |

인터페이스는 `If_SeatSwitch`, `If_DialLed`(Sender-Receiver)와 `If_Heating`, `If_Setting`(Client-Server)로 정의되어 있고, 하드웨어 쪽은 `If_Pwm`, `If_DigDir`, `If_AnaInDir`로 IoHwAb에 연결된다.

## 동작 흐름

그림의 연결을 따라가면 다음 순서로 동작한다.

1. **스위치 입력** : SeatSwitch가 CAN으로 열선 시트 명령을 받으면, Sender-Receiver로 SHControl에 스위치 상태를 보낸다. 데이터를 받으면 `Re_SHSwitch`가 실행된다.
2. **다이얼 값 확인** : SHControl은 Client-Server(`Setting`)로 HeatingDial에 현재 다이얼 위치를 요청한다. HeatingDial은 10ms마다 포텐셔미터 ADC 값을 읽어 두고 있다.
3. **열선 출력** : SHControl이 Client-Server(`HeatingElement`)로 SeatHeating을 호출하면, 서버 Runnable `Re_SeatHeating`이 PWM 듀티를 바꿔 LED4 밝기(열선 강도)를 조절한다.
4. **상태 표시** : SHControl은 `DialLed`를 Queue 방식 Sender-Receiver로 보내고, HeatingDial이 LED로 상태를 표시한다. 동작 확인용 Alive LED도 주기적으로 토글한다.
5. **상태 보고** : 현재 상태는 SeatSwitch의 RcStatus를 통해 다시 CAN으로 송신된다.

## 이 구조에서 배운 점

- **SHControl은 하드웨어를 전혀 모른다.** 핀 번호, ADC 채널, PWM 타이머가 바뀌어도 SHControl 코드는 그대로다. 바뀌는 것은 IoHwAb 아래의 설정뿐이다.
- 주기적으로 바뀌는 값(스위치 상태)은 **Sender-Receiver**, 필요할 때 결과를 받아야 하는 동작(다이얼 위치 읽기, 출력 설정)은 **Client-Server**로 나눈 것이 핵심이다.
- 이벤트를 하나도 놓치면 안 되는 데이터는 **Queued** Sender-Receiver로, 최신 값만 중요하면 Last-is-best로 설정한다.

신호가 CAN까지 내려가는 과정은 [Autosar (6) : Communication Layers 실습](/2025-03-19-autosar6/)에 정리했다.

