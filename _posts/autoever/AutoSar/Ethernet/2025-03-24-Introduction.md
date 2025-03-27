---
layout: post
title:  "Ehternet (1) : Ehternet "
date:   2025-03-19 +09001
categories: Autoever
tag: "ethernet"
---

두가닥의 선으로 통신을 가능하게 한 것이 can 통신

- CAN 통신이 맞나
예측 불가능한 지연
deadline 내에 끝낼 수 있는가. Yes = 실시간 통신 RUNTIME

## CAN Node의 구조와 기능 정리

하나의 CAN Node는 다음과 같은 3가지 주요 구성 요소로 이루어져 있습니다.

``` text
[ Application (Host) ]
         ↓
[ CAN Controller ]
         ↓
[ CAN Transceiver ]
         ↓
[ CAN Bus ]
```

---

### 1. Host

**설명**  

- Application 코드 및 AUTOSAR 통신 스택이 위치한 영역
- CAN 통신을 위한 데이터 생성 및 처리 수행

**기능**  

- CAN 메시지 생성 (`Rte_Write`, `CanIf_Transmit` 등)
- 수신 메시지 처리 (`Rte_Read`, 신호 해석 등)
- AUTOSAR 모듈들 (SWC, RTE, COM, PduR 등) 포함

---

### 2. CAN Controller

**설명**  

- MCU 내부에 존재하는 하드웨어 모듈
- CAN 프로토콜 처리 담당

**기능**  

- 프레임 인코딩/디코딩 (ID, DLC, CRC, ACK 등)
- Tx/Rx Mailbox 관리
- 에러 감지 및 상태 제어 (Bus-Off, Passive 등)
- Bit Timing 및 Baudrate 설정

---

### 3. CAN Transceiver

**설명**  

- 디지털 신호 ↔ 전기 신호를 변환하는 외부 칩
- CAN Controller와 물리적 CAN Bus 사이 연결

**기능**  

- 디지털 신호 → Differential Signal (CAN_H / CAN_L)
- 수신 전압 차 → 디지털 신호 복원
- 버스 보호 (ESD, 전압 제한, 절연 등)
- 예: TJA1040, MCP2551, TJA1051 등

---

## 송수신 흐름 예시

### 송신 흐름

1. Host에서 메시지 생성
2. CAN Controller가 CAN 프레임 생성
3. Transceiver가 전기 신호로 변환
4. CAN Bus를 통해 전송

### 수신 흐름

1. Transceiver가 신호를 수신하여 디지털화
2. CAN Controller가 프레임 디코딩
3. Host에 메시지 전달 (Application 처리)

---

| 구성 요소         | 위치         | 주요 기능 |
|------------------|--------------|-----------|
| **Host**         | MCU 내 Application 영역 | 데이터 처리, AUTOSAR 통신 스택 |
| **CAN Controller** | MCU 내부 IP 모듈 | 프로토콜 처리, 프레임 제어 |
| **CAN Transceiver** | 외부 IC | 전기적 신호 변환, 보호 기능 |

- CAN FD (Flexible Datarate)

- Flexray
10Ms 속도
현대는 사용하지 않고 고급차들이 많이 사용

- Ehernet
Ethernet Rj-45
BroadR-Reach
=> Ethernet AVB

- Powertrain
일반적으로 10ms로 CAN 통신해서 RPM의 값을 전송한다.

- ADAS(Advanced Driver Assistance System)
대규모 센서들이 많기 때문에 **대용량 데이터 전송**과 **실시간 데이터 전송**이 둘 다 중요한 도매인이다. 그러나 Linex 기반 운영체제를 사용하기 때문에 End to End Latency는 250us~1ms로 긴시간이 필요하다. 20-100Mbps bandwidth가 필요하고 통신속도는 1~500ms까지 소요된다.

### 차량 E/E Architecture

SDV로의 변화를 위해 차량 E/E를 단계적으로 변화를 주고 있다.

1. Distributed
　전통적인 분산시스템을 가진 아키텍쳐

2. Domain centralized
　기존 제어기는 단순 센서 및 엑추에이터 기능을 수행하고 통합 제어 등을 DCU에서 수행하는 형태로 변화가 필요함
일반적으로 기존의 도메인ㅇ서 가지고 있는 기능 중 기존의 ECU는 단순 센서 및 액추에이터 역할을 수행하고 통합제어가 필요한 기능은 DCU에서 수행함
DCU간에 고속의 Ethernet으로 연결되어 다양한 도메인의 정보를 빠르게 처리할 수 있음
기존 제어기 대비 고성능의 CPU, 필요시 GPU도 탑재

3. Vehicle centralized
　고성능 Vehicle computer가 지역별로 배치된 Zone Control Unit의 정보를 수집하여 통합 제어

Vehicle Centralized Architecture이랑 Zonal Architecture을 같이 사용한다.
Central Computer은 일반적으로 2대 이상있다.
main 1개, backup 및 sub 1 + @ 개

Event Trigger vs Time triggered
Deterministck 결정 하능한 경쟁이 좋다.




## 262626

위험, 보안, 문제가 생길 수 있느 부분을 보완할 수 있는 메커니즘이 정의되어 있다.

## ECC

데이터 전송 또는 저장 시 발생할 ㅅ ㅜ있는 오류를 검출하려고 수정할 수 있는 코드를 의미
데이터 무결성 보장

## Lockstep

 두 개 이상의 프로세서가 같은 명령어를 동시에 실행하여 결과를 비교하고 일치 여부로 오류를 검출

## Hamming Code